
```dataviewjs
// Configuration
const CURRENT_FOLDER = dv.current().file.folder;
const IGNORED_FILES = ["_Task Template"];
const DONE_STATUSES = ["done", "complete", "completed", "finished", "✅", "true", true];
const YELLOW_THRESHOLD = -0.10;
const RED_THRESHOLD = -0.10;

// ========== DEBUG MODE ==========
const DEBUG = false;

function debug(label, data) {
    if (!DEBUG) return;
    console.log(`[TaskDashboard] ${label}:`, data);
}

// Debug: Show all pages in vault
if (DEBUG) {
    debug("Dashboard folder", CURRENT_FOLDER);
    const allPages = dv.pages();
    debug("Total pages in vault", allPages.length);
    const tasksInFolder = dv.pages().where(p => {
        const isTask = p["task-start"] && p["task-due"]
        const folder = p.file.folder;
        return isTask && (folder === CURRENT_FOLDER || folder.startsWith(CURRENT_FOLDER + "/"));
    });
    debug("Pages in folder (including subfolders)", tasksInFolder.length);
    // Render debug info on page
    dv.header(3, "🔧 Debug Info");
    dv.paragraph(`Dashboard folder: "${CURRENT_FOLDER}"`);
    dv.paragraph(`Tasks in this folder (recursive): ${tasksInFolder.length}`);
    dv.paragraph("---");
}

// Parse subtask table from page content
async function parseSubtasks(page) {
    const content = await dv.io.load(page.file.path);
    const lines = content.split('\n');
    // Find table lines (look for markdown table syntax)
    const tableLines = [];
    let inTable = false;
    for (const line of lines) {
        const trimmed = line.trim();
        if (trimmed.startsWith('|') && trimmed.endsWith('|')) {
            inTable = true;
            tableLines.push(trimmed);
        } else if (inTable && trimmed === '') {
            break;
        } else if (inTable && !trimmed.startsWith('|')) {
            break;
        }
    }
    // Need at least header + separator + one data row
    if (tableLines.length < 3) return [];
    // Parse header to find column indices
    const headerCells = tableLines[0]
        .split('|')
        .map(s => s.trim().toLowerCase())
        .slice(1, -1);  // Remove first/last empty elements from pipe split, preserve empty cells
    const pointsIdx = headerCells.indexOf('points');
    const statusIdx = headerCells.indexOf('status');
    if (pointsIdx === -1 || statusIdx === -1) return [];
    // Parse data rows (skip header at index 0 and separator at index 1)
    const subtasks = [];
    for (let i = 2; i < tableLines.length; i++) {
        const cells = tableLines[i]
            .split('|')
            .map(s => s.trim())
            .slice(1, -1);  // Remove first/last empty elements from pipe split, preserve empty cells
        if (cells.length > Math.max(pointsIdx, statusIdx)) {
            const points = parseFloat(cells[pointsIdx]) || 0;
            const status = cells[statusIdx].toLowerCase().trim();
            subtasks.push({ points, status });
        }
    }
    return subtasks;
}

// Calculate schedule status
function calculateScheduleStatus(startDate, dueDate, actualProgress) {
    const now = new Date();
    now.setHours(0, 0, 0, 0);
    if (!startDate || !dueDate || dueDate <= startDate) {
        return { expectedProgress: actualProgress, progressDiff: 0 };
    }
    const start = new Date(startDate);
    const due = new Date(dueDate);
    start.setHours(0, 0, 0, 0);
    due.setHours(0, 0, 0, 0);
    const totalDuration = due - start;
    const elapsed = now - start;
    let expectedProgress;
    if (elapsed <= 0) {
        expectedProgress = 0;
    } else if (elapsed >= totalDuration) {
        expectedProgress = 1;
    } else {
        expectedProgress = elapsed / totalDuration;
    }
    const progressDiff = actualProgress - expectedProgress;
    return { expectedProgress, progressDiff };
}

// Determine color based on progress difference
function getStatusColor(progressDiff, actualProgress) {
    if (actualProgress === 0 && progressDiff >= 0) return { color: '#6b7280', label: 'Unstarted' };
    if (progressDiff >= 0) return { color: '#22c55e', label: 'On Track' };
    if (progressDiff >= YELLOW_THRESHOLD) return { color: '#eab308', label: 'Slightly Behind' };
    return { color: '#ef4444', label: 'Behind Schedule' };
}

// Format date for display
function formatDate(date) {
    if (!date) return 'N/A';
    const d = new Date(date);
    return d.toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' });
}

// Main execution
debug("Using folder", CURRENT_FOLDER);

const pages = dv.pages().where(p => {
    const folder = p.file.folder;
    const isInFolder = folder === CURRENT_FOLDER ||
        folder.startsWith(CURRENT_FOLDER + "/");
    const isTask = p["task-start"] && p["task-due"];
    const isIgnored = IGNORED_FILES.includes(p.file.name);
    return isInFolder && isTask && !isIgnored;
});

debug("Pages found after filter", pages.length);
if (DEBUG && pages.length > 0) {
    debug("Found pages", pages.map(p => p.file.path).array());
}

// Get parent folder name from a file path
function getParentFolder(filePath) {
    const parts = filePath.split('/');
    // Return the immediate parent folder (second to last part)
    return parts.length >= 2 ? parts[parts.length - 2] : 'Root';
}

const taskData = await Promise.all(pages.map(async (page) => {
    const subtasks = await parseSubtasks(page);
    const totalPoints = subtasks.reduce((sum, s) => sum + s.points, 0);
    const completedPoints = subtasks
        .filter(s => DONE_STATUSES.includes(s.status))
        .reduce((sum, s) => sum + s.points, 0);
    const actualProgress = totalPoints > 0 ? completedPoints / totalPoints : 0;
    const { expectedProgress, progressDiff } = calculateScheduleStatus(
        page["task-start"],
        page["task-due"],
        actualProgress
    );
    const statusInfo = getStatusColor(progressDiff, actualProgress);
    const parentFolder = getParentFolder(page.file.path);
    const dueDate = page["task-due"];
    const now = new Date();
    now.setHours(0, 0, 0, 0);
    const due = new Date(dueDate);
    due.setHours(0, 0, 0, 0);
    const daysRemaining = Math.ceil((due - now) / (1000 * 60 * 60 * 24));
    return {
        name: page.file.name,
        link: page.file.link,
        totalPoints,
        completedPoints,
        actualProgress,
        expectedProgress,
        progressDiff,
        statusColor: statusInfo.color,
        statusLabel: statusInfo.label,
        startDate: page["task-start"],
        dueDate,
        daysRemaining,
        parentFolder
    };
}));

// Group tasks by parent folder, then sort within each group by progressDiff
const groupedTasks = {};
for (const task of taskData) {
    if (!groupedTasks[task.parentFolder]) {
        groupedTasks[task.parentFolder] = [];
    }
    groupedTasks[task.parentFolder].push(task);
}

// Sort tasks within each group by progressDiff (most behind first)
for (const folder in groupedTasks) {
    groupedTasks[folder].sort((a, b) => a.progressDiff - b.progressDiff);
}

// Sort folder names alphabetically
const sortedFolders = Object.keys(groupedTasks).sort();

// Render dashboard
if (taskData.length === 0) {
    dv.paragraph("*No tasks found in the Tasks folder.*");
} else {
    const container = dv.el('div', '', { cls: 'task-dashboard' });
    // Inject CSS to hide details marker
    if (!document.getElementById('task-dashboard-style')) {
        const style = document.createElement('style');
        style.id = 'task-dashboard-style';
        style.textContent = `
            .task-dashboard details summary { list-style: none !important; }
            .task-dashboard details summary::-webkit-details-marker { display: none !important; }
            .task-dashboard details summary::marker { display: none !important; content: none !important; }
        `;
        document.head.appendChild(style);
    }
    for (const folderName of sortedFolders) {
        const details = dv.el('details', '', {
            container,
            attr: { style: 'margin-top: 16px;', open: true }
        });
        dv.el('summary', `📁 ${folderName} (${groupedTasks[folderName].length})`, {
            container: details,
            attr: { style: 'cursor: pointer; font-size: 1.2em; font-weight: bold; margin-bottom: 12px; padding: 8px 0; border-bottom: 1px solid var(--background-modifier-border);' }
        });
        for (const task of groupedTasks[folderName]) {
            const pctActual = Math.round(task.actualProgress * 100);
            const pctExpected = Math.round(task.expectedProgress * 100);
            const diffPct = Math.round(task.progressDiff * 100);
            const diffSign = diffPct >= 0 ? '+' : '';
            const card = dv.el('div', '', {
                container: details,
                attr: {
                    style: 'background: var(--background-secondary); border-radius: 8px; padding: 16px; margin-bottom: 16px; border-left: 4px solid ' + task.statusColor
                }
            });
            const header = dv.el('div', '', {
                container: card,
                attr: { style: 'display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px; gap: 8px;' }
            });
            const headerLeft = dv.el('div', '', {
                container: header,
                attr: { style: 'display: flex; align-items: center; justify-content: flex-start; gap: 8px; flex-wrap: wrap; flex: 1;' }
            });
            dv.el('strong', task.link, { container: headerLeft });
            dv.el('span', `📁 ${task.parentFolder}`, {
                container: headerLeft,
                attr: {
                    style: 'background: var(--background-modifier-border); color: var(--text-muted); padding: 2px 8px; border-radius: 4px; font-size: 0.75em;'
                }
            });
            dv.el('span', task.statusLabel, {
                container: header,
                attr: {
                    style: `background: ${task.statusColor}; color: white; padding: 2px 8px; border-radius: 4px; font-size: 0.85em; white-space: nowrap;`
                }
            });
            const barContainer = dv.el('div', '', {
                container: card,
                attr: {
                    style: 'background: var(--background-primary); border-radius: 4px; height: 24px; position: relative; overflow: hidden; margin: 8px 0;'
                }
            });
            dv.el('div', '', {
                container: barContainer,
                attr: {
                    style: `background: ${task.statusColor}; height: 100%; width: ${pctActual}%; transition: width 0.3s ease;`
                }
            });
            dv.el('div', '', {
                container: barContainer,
                attr: {
                    style: `position: absolute; top: 0; left: ${pctExpected}%; width: 3px; height: 100%; background: var(--text-muted); opacity: 0.7;`,
                    title: `Expected: ${pctExpected}%`
                }
            });
            dv.el('div', `${pctActual}%`, {
                container: barContainer,
                attr: {
                    style: 'position: absolute; top: 0; left: 0; right: 0; bottom: 0; display: flex; align-items: center; justify-content: center; font-weight: bold; font-size: 0.85em; color: var(--text-normal); mix-blend-mode: difference;'
                }
            });
            const stats = dv.el('div', '', {
                container: card,
                attr: { style: 'display: flex; gap: 16px; font-size: 0.85em; color: var(--text-muted); flex-wrap: wrap;' }
            });
            dv.el('span', `📊 ${task.completedPoints}/${task.totalPoints} pts`, { container: stats });
            dv.el('span', `📅 Due: ${formatDate(task.dueDate)}`, { container: stats });
            dv.el('span', `📍 Expected: ${pctExpected}%`, { container: stats });
            dv.el('span', `${diffSign}${diffPct}% vs schedule`, {
                container: stats,
                attr: { style: `color: ${task.statusColor}; font-weight: bold;` }
            });
            const daysLabel = task.daysRemaining === 1 ? 'day' : 'days';
            const daysText = task.daysRemaining < 0 
                ? `${Math.abs(task.daysRemaining)} ${daysLabel} overdue`
                : `${task.daysRemaining} ${daysLabel} remaining`;
            dv.el('span', `⏳ ${daysText}`, { container: stats });
        }
    }
}
```
