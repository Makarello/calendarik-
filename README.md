<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
    <title>Календарь смен — локальная версия (адаптивная)</title>
    <!-- FullCalendar CSS + JS -->
    <link href="https://cdn.jsdelivr.net/npm/fullcalendar@5.11.3/main.min.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/fullcalendar@5.11.3/main.min.js"></script>
    <style>
        * {
            box-sizing: border-box;
        }
        body {
            font-family: system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', sans-serif;
            margin: 0;
            padding: 12px;
            background-color: #e9eef3;
        }
        /* Панель проектов — адаптивная */
        .project-bar {
            max-width: 1400px;
            margin: 0 auto 15px;
            background: white;
            padding: 12px 16px;
            border-radius: 20px;
            box-shadow: 0 2px 8px rgba(0,0,0,0.05);
            display: flex;
            flex-wrap: wrap;
            gap: 10px;
            align-items: center;
        }
        .project-bar select {
            flex: 3;
            min-width: 150px;
            padding: 10px 12px;
            font-size: 16px;
            border: 1px solid #ccc;
            border-radius: 30px;
            background-color: white;
        }
        .project-bar button {
            flex: 1;
            padding: 10px 12px;
            border: none;
            border-radius: 30px;
            cursor: pointer;
            font-size: 14px;
            font-weight: 600;
            background-color: #2c7da0;
            color: white;
            transition: 0.2s;
            white-space: nowrap;
        }
        .project-bar button.delete {
            background-color: #d9534f;
        }
        .project-bar button.export-btn {
            background-color: #5cb85c;
        }
        .project-bar button.import-btn {
            background-color: #f0ad4e;
        }
        .project-bar button:hover {
            opacity: 0.85;
            transform: scale(0.98);
        }
        /* Календарь */
        #calendar {
            max-width: 1400px;
            margin: 0 auto;
            background: white;
            padding: 12px;
            border-radius: 24px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.1);
        }
        /* Настройка для мобильных: шрифты и отступы */
        .fc {
            font-size: 0.85rem;
        }
        @media (max-width: 768px) {
            .fc .fc-toolbar {
                flex-direction: column;
                gap: 10px;
            }
            .fc .fc-toolbar-title {
                font-size: 1.2rem;
            }
            .fc .fc-button {
                padding: 0.3rem 0.6rem;
                font-size: 0.8rem;
            }
            .project-bar button {
                font-size: 12px;
                padding: 8px 10px;
            }
        }
        /* Модальное окно */
        .modal {
            display: none;
            position: fixed;
            z-index: 1000;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            background-color: rgba(0,0,0,0.5);
            justify-content: center;
            align-items: center;
        }
        .modal-content {
            background: white;
            padding: 25px;
            border-radius: 28px;
            width: 90%;
            max-width: 400px;
            box-shadow: 0 10px 25px rgba(0,0,0,0.2);
        }
        .modal-content h3 {
            margin-top: 0;
            color: #1e2a3a;
        }
        .form-group {
            margin-bottom: 18px;
        }
        .form-group label {
            display: block;
            margin-bottom: 6px;
            font-weight: 600;
            color: #0a2f44;
        }
        .form-group input, .form-group select {
            width: 100%;
            padding: 10px;
            border: 1px solid #ccd7e4;
            border-radius: 40px;
            font-size: 16px;
        }
        .button-group {
            display: flex;
            justify-content: flex-end;
            gap: 12px;
            margin-top: 20px;
        }
        .btn-primary {
            background-color: #2c7da0;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 40px;
            font-weight: bold;
        }
        .btn-cancel {
            background-color: #adb5bd;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 40px;
        }
        /* кнопка импорта (скрытый инпут) */
        #importFileInput {
            display: none;
        }
    </style>
</head>
<body>

<div class="project-bar">
    <select id="projectSelect"></select>
    <button id="newProjectBtn">➕ Проект</button>
    <button id="deleteProjectBtn" class="delete">🗑️ Удалить</button>
    <button id="exportDataBtn" class="export-btn">💾 Экспорт</button>
    <button id="importDataBtn" class="import-btn">📂 Импорт</button>
    <input type="file" id="importFileInput" accept=".json">
</div>

<div id="calendar"></div>

<!-- Модальное окно добавления смены -->
<div id="shiftModal" class="modal">
    <div class="modal-content">
        <h3 id="modalTitle">Добавить смену</h3>
        <div class="form-group">
            <label>Сотрудник</label>
            <select id="employeeSelect">
                <option value="Иван Петров">Иван Петров</option>
                <option value="Мария Смирнова">Мария Смирнова</option>
                <option value="Алексей Иванов">Алексей Иванов</option>
                <option value="Елена Кузнецова">Елена Кузнецова</option>
                <option value="Дмитрий Соколов">Дмитрий Соколов</option>
            </select>
        </div>
        <div class="form-group">
            <label>Начало</label>
            <input type="datetime-local" id="startTime" required>
        </div>
        <div class="form-group">
            <label>Конец</label>
            <input type="datetime-local" id="endTime" required>
        </div>
        <div class="button-group">
            <button type="button" class="btn-cancel" onclick="closeModal()">Отмена</button>
            <button type="button" class="btn-primary" onclick="saveShift()">Сохранить</button>
        </div>
    </div>
</div>

<script>
    // ---- ГЛОБАЛЬНЫЕ ПЕРЕМЕННЫЕ ----
    let calendar;
    let projects = [];          // [{ id, name, shifts }]
    let activeProjectId = null;
    const STORAGE_KEY = 'shiftCalendarProjects';

    // DOM элементы
    const projectSelect = document.getElementById('projectSelect');
    const newProjectBtn = document.getElementById('newProjectBtn');
    const deleteProjectBtn = document.getElementById('deleteProjectBtn');
    const exportDataBtn = document.getElementById('exportDataBtn');
    const importDataBtn = document.getElementById('importDataBtn');
    const importFileInput = document.getElementById('importFileInput');
    const modal = document.getElementById('shiftModal');
    const employeeSelect = document.getElementById('employeeSelect');
    const startTimeInput = document.getElementById('startTime');
    const endTimeInput = document.getElementById('endTime');
    const modalTitle = document.getElementById('modalTitle');

    let selectedStart = null, selectedEnd = null;

    // ---- ЦВЕТ ПО СОТРУДНИКУ ----
    function getRandomColor(name) {
        let hash = 0;
        for (let i = 0; i < name.length; i++) {
            hash = name.charCodeAt(i) + ((hash << 5) - hash);
        }
        const hue = hash % 360;
        return `hsl(${hue}, 70%, 85%)`;
    }

    // ---- LOCALSTORAGE ----
    function saveToLocalStorage() {
        localStorage.setItem(STORAGE_KEY, JSON.stringify(projects));
    }

    function loadFromLocalStorage() {
        const saved = localStorage.getItem(STORAGE_KEY);
        if (saved) {
            try {
                projects = JSON.parse(saved);
                // Валидация
                if (!Array.isArray(projects)) projects = [];
                projects = projects.filter(p => p && p.id && p.name && Array.isArray(p.shifts));
            } catch(e) { projects = []; }
        }
        if (!projects.length) {
            // Дефолтный проект с демо-сменами (сегодня+1, сегодня+2)
            const now = new Date();
            const tomorrow = new Date(now);
            tomorrow.setDate(now.getDate() + 1);
            tomorrow.setHours(9, 0, 0);
            const tomorrowEnd = new Date(tomorrow);
            tomorrowEnd.setHours(17, 0, 0);

            const dayAfter = new Date(now);
            dayAfter.setDate(now.getDate() + 2);
            dayAfter.setHours(14, 0, 0);
            const dayAfterEnd = new Date(dayAfter);
            dayAfterEnd.setHours(22, 0, 0);

            projects = [{
                id: 'proj_' + Date.now(),
                name: 'Основной проект',
                shifts: [
                    {
                        id: 'demo1',
                        title: 'Иван Петров',
                        start: tomorrow.toISOString(),
                        end: tomorrowEnd.toISOString(),
                        backgroundColor: getRandomColor('Иван Петров'),
                        employee: 'Иван Петров'
                    },
                    {
                        id: 'demo2',
                        title: 'Мария Смирнова',
                        start: dayAfter.toISOString(),
                        end: dayAfterEnd.toISOString(),
                        backgroundColor: getRandomColor('Мария Смирнова'),
                        employee: 'Мария Смирнова'
                    }
                ]
            }];
            saveToLocalStorage();
        }
        if (!activeProjectId || !projects.find(p => p.id === activeProjectId)) {
            activeProjectId = projects[0].id;
        }
    }

    // ---- ОТРИСОВКА UI (список проектов + календарь) ----
    function renderUI() {
        if (!projects.length) {
            projectSelect.innerHTML = '<option>Нет проектов</option>';
            projectSelect.disabled = true;
            deleteProjectBtn.disabled = true;
            calendar.removeAllEvents();
            return;
        }
        projectSelect.disabled = false;
        deleteProjectBtn.disabled = false;
        // заполняем select
        projectSelect.innerHTML = '';
        projects.forEach(p => {
            const option = document.createElement('option');
            option.value = p.id;
            option.textContent = p.name;
            if (p.id === activeProjectId) option.selected = true;
            projectSelect.appendChild(option);
        });
        // отображаем смены активного проекта
        const activeProject = projects.find(p => p.id === activeProjectId);
        if (activeProject) {
            calendar.removeAllEvents();
            calendar.addEventSource(activeProject.shifts);
        } else {
            calendar.removeAllEvents();
        }
    }

    // ---- УПРАВЛЕНИЕ ПРОЕКТАМИ ----
    function switchProject(projectId) {
        if (!projects.find(p => p.id === projectId)) return;
        activeProjectId = projectId;
        renderUI();
    }

    function createNewProject() {
        const name = prompt('Название проекта:', 'Новый проект');
        if (!name) return;
        const newId = 'proj_' + Date.now() + '_' + Math.random().toString(36).substr(2, 6);
        projects.push({
            id: newId,
            name: name,
            shifts: []
        });
        activeProjectId = newId;
        saveToLocalStorage();
        renderUI();
    }

    function deleteCurrentProject() {
        if (projects.length <= 1) {
            alert('Нельзя удалить единственный проект.');
            return;
        }
        const projectName = projects.find(p => p.id === activeProjectId)?.name;
        if (!confirm(`Удалить проект "${projectName}"? Все его смены исчезнут.`)) return;
        const index = projects.findIndex(p => p.id === activeProjectId);
        if (index !== -1) {
            projects.splice(index, 1);
            activeProjectId = projects[0].id;
            saveToLocalStorage();
            renderUI();
        }
    }

    // ---- ЭКСПОРТ / ИМПОРТ (для переноса данных между устройствами) ----
    function exportData() {
        const dataStr = JSON.stringify(projects, null, 2);
        const blob = new Blob([dataStr], {type: 'application/json'});
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `shift-calendar-backup-${new Date().toISOString().slice(0,19)}.json`;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
    }

    function importData(file) {
        const reader = new FileReader();
        reader.onload = function(e) {
            try {
                const imported = JSON.parse(e.target.result);
                if (Array.isArray(imported) && imported.every(p => p.id && p.name && Array.isArray(p.shifts))) {
                    projects = imported;
                    if (!projects.length) throw new Error('Пустой массив');
                    activeProjectId = projects[0].id;
                    saveToLocalStorage();
                    renderUI();
                    alert('Импорт успешен! Все данные заменены.');
                } else {
                    throw new Error('Неверный формат');
                }
            } catch(err) {
                alert('Ошибка: неверный JSON-файл. Загрузите файл, созданный через "Экспорт".');
            }
        };
        reader.readAsText(file);
    }

    // ---- МОДАЛЬНОЕ ОКНО ДЛЯ ДОБАВЛЕНИЯ СМЕНЫ ----
    function openModal(start, end) {
        selectedStart = start;
        selectedEnd = end;
        const format = (date) => {
            const d = new Date(date);
            return d.toISOString().slice(0,16);
        };
        startTimeInput.value = format(start);
        endTimeInput.value = format(end);
        modalTitle.innerText = 'Добавить смену';
        modal.style.display = 'flex';
    }

    window.closeModal = function() {
        modal.style.display = 'none';
    };

    function saveShift() {
        const employee = employeeSelect.value;
        const start = startTimeInput.value;
        const end = endTimeInput.value;

        if (!employee || !start || !end) {
            alert('Заполните все поля');
            return;
        }
        if (new Date(start) >= new Date(end)) {
            alert('Начало должно быть раньше конца');
            return;
        }

        const project = projects.find(p => p.id === activeProjectId);
        if (!project) return;

        const newEvent = {
            id: Date.now() + '_' + Math.random().toString(36).substr(2, 8),
            title: employee,
            start: start,
            end: end,
            backgroundColor: getRandomColor(employee),
            borderColor: '#2c7da0',
            textColor: '#000',
            employee: employee
        };
        project.shifts.push(newEvent);
        saveToLocalStorage();
        renderUI();  // обновит календарь
        closeModal();
    }

    window.deleteShift = function(eventId) {
        if (!confirm('Удалить смену?')) return;
        const project = projects.find(p => p.id === activeProjectId);
        if (!project) return;
        const idx = project.shifts.findIndex(s => s.id === eventId);
        if (idx !== -1) {
            project.shifts.splice(idx, 1);
            saveToLocalStorage();
            renderUI();
        }
    };

    // ---- ИНИЦИАЛИЗАЦИЯ ----
    document.addEventListener('DOMContentLoaded', function() {
        const calendarEl = document.getElementById('calendar');
        calendar = new FullCalendar.Calendar(calendarEl, {
            initialView: 'timeGridWeek',
            headerToolbar: {
                left: 'prev,next today',
                center: 'title',
                right: 'dayGridMonth,timeGridWeek,timeGridDay'
            },
            locale: 'ru',
            selectable: true,
            selectMirror: true,
            editable: false,
            dayMaxEvents: true,
            weekends: true,
            select: function(info) {
                openModal(info.start, info.end);
            },
            eventClick: function(info) {
                deleteShift(info.event.id);
            },
            // адаптивная высота
            height: 'auto',
            aspectRatio: 1.8,
        });
        calendar.render();

        // Загружаем данные
        loadFromLocalStorage();
        renderUI();

        // Обработчики
        newProjectBtn.addEventListener('click', createNewProject);
        deleteProjectBtn.addEventListener('click', deleteCurrentProject);
        projectSelect.addEventListener('change', (e) => switchProject(e.target.value));
        exportDataBtn.addEventListener('click', exportData);
        importDataBtn.addEventListener('click', () => importFileInput.click());
        importFileInput.addEventListener('change', (e) => {
            if (e.target.files.length) importData(e.target.files[0]);
            importFileInput.value = '';
        });
    });
</script>
</body>
</html>
