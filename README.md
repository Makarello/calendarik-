<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes">
    <title>Календарь смен — local версия</title>
    <!-- FullCalendar from CDN (международный, работает всегда) -->
    <link href="https://cdn.jsdelivr.net/npm/fullcalendar@5.11.3/main.min.css" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/fullcalendar@5.11.3/main.min.js"></script>
    <style>
        * {
            box-sizing: border-box;
        }
        body {
            margin: 0;
            padding: 16px;
            background: #e2e8f0;
            font-family: system-ui, -apple-system, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
        }
        /* Контейнеры */
        .app-container {
            max-width: 1400px;
            margin: 0 auto;
        }
        /* Панель проектов */
        .project-bar {
            background: white;
            border-radius: 32px;
            padding: 12px 20px;
            margin-bottom: 24px;
            display: flex;
            flex-wrap: wrap;
            gap: 12px;
            align-items: center;
            box-shadow: 0 4px 12px rgba(0,0,0,0.05);
        }
        .project-bar select {
            flex: 3;
            min-width: 150px;
            padding: 12px 16px;
            font-size: 1rem;
            border: 1px solid #cbd5e1;
            border-radius: 60px;
            background: white;
            font-weight: 500;
        }
        .project-bar button {
            padding: 10px 18px;
            border: none;
            border-radius: 40px;
            font-weight: 600;
            font-size: 0.9rem;
            cursor: pointer;
            transition: all 0.2s ease;
            color: white;
            background-color: #2c7da0;
            box-shadow: 0 1px 2px rgba(0,0,0,0.05);
        }
        .project-bar button.delete {
            background-color: #c0392b;
        }
        .project-bar button.export {
            background-color: #27ae60;
        }
        .project-bar button.import {
            background-color: #f39c12;
        }
        .project-bar button:hover {
            opacity: 0.85;
            transform: translateY(-1px);
        }
        /* Календарь */
        #calendar {
            background: white;
            border-radius: 32px;
            padding: 16px;
            box-shadow: 0 8px 20px rgba(0,0,0,0.08);
        }
        /* Мобильная адаптация */
        @media (max-width: 700px) {
            body {
                padding: 10px;
            }
            .project-bar button {
                padding: 8px 14px;
                font-size: 0.8rem;
            }
            .fc .fc-toolbar {
                flex-direction: column;
                gap: 8px;
            }
            .fc .fc-toolbar-title {
                font-size: 1.2rem;
            }
            .fc .fc-button {
                padding: 0.3rem 0.6rem;
                font-size: 0.8rem;
            }
        }
        /* Модалка */
        .modal {
            display: none;
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0,0,0,0.5);
            justify-content: center;
            align-items: center;
            z-index: 2000;
        }
        .modal-card {
            background: white;
            max-width: 420px;
            width: 90%;
            border-radius: 40px;
            padding: 24px;
            box-shadow: 0 20px 35px rgba(0,0,0,0.2);
        }
        .modal-card h3 {
            margin: 0 0 16px 0;
            font-size: 1.6rem;
            font-weight: 600;
        }
        .form-group {
            margin-bottom: 18px;
        }
        .form-group label {
            display: block;
            margin-bottom: 6px;
            font-weight: 600;
            color: #1e293b;
        }
        .form-group select, .form-group input {
            width: 100%;
            padding: 12px 14px;
            border: 1px solid #cbd5e1;
            border-radius: 60px;
            font-size: 1rem;
        }
        .buttons {
            display: flex;
            justify-content: flex-end;
            gap: 12px;
            margin-top: 20px;
        }
        .btn-save {
            background: #2c7da0;
            color: white;
            border: none;
            padding: 10px 22px;
            border-radius: 40px;
            font-weight: bold;
        }
        .btn-cancel {
            background: #94a3b8;
            color: white;
            border: none;
            padding: 10px 22px;
            border-radius: 40px;
        }
    </style>
</head>
<body>
<div class="app-container">
    <div class="project-bar">
        <select id="projectSelect"></select>
        <button id="newProjectBtn">➕ Новый</button>
        <button id="deleteProjectBtn" class="delete">🗑️ Удалить</button>
        <button id="exportBtn" class="export">📁 Экспорт</button>
        <button id="importBtn" class="import">📂 Импорт</button>
    </div>
    <div id="calendar"></div>
</div>

<!-- Модалка добавления смены -->
<div id="shiftModal" class="modal">
    <div class="modal-card">
        <h3>Новая смена</h3>
        <div class="form-group">
            <label>Сотрудник</label>
            <select id="employeeSelect">
                <option value="Анна Козлова">Анна Козлова</option>
                <option value="Борис Лебедев">Борис Лебедев</option>
                <option value="Виктор Смирнов">Виктор Смирнов</option>
                <option value="Галина Петрова">Галина Петрова</option>
                <option value="Дмитрий Орлов">Дмитрий Орлов</option>
            </select>
        </div>
        <div class="form-group">
            <label>Начало</label>
            <input type="datetime-local" id="startInput" required>
        </div>
        <div class="form-group">
            <label>Конец</label>
            <input type="datetime-local" id="endInput" required>
        </div>
        <div class="buttons">
            <button class="btn-cancel" id="closeModalBtn">Отмена</button>
            <button class="btn-save" id="saveShiftBtn">Сохранить</button>
        </div>
    </div>
</div>

<script>
    // ----------------------------------------------
    // 1. РАБОТА С LOCALSTORAGE
    // ----------------------------------------------
    let projects = [];
    let activeProjectId = null;
    const STORAGE_KEY = 'shiftCalendarUltimate';

    function saveToLocal() {
        localStorage.setItem(STORAGE_KEY, JSON.stringify(projects));
    }

    function loadFromLocal() {
        const raw = localStorage.getItem(STORAGE_KEY);
        if (raw) {
            try {
                const parsed = JSON.parse(raw);
                if (Array.isArray(parsed) && parsed.length && parsed[0].id && parsed[0].shifts) {
                    projects = parsed;
                } else {
                    throw new Error('invalid');
                }
            } catch(e) {
                createDefaultProjects();
            }
        } else {
            createDefaultProjects();
        }
        if (!activeProjectId || !projects.find(p => p.id === activeProjectId)) {
            activeProjectId = projects[0].id;
        }
    }

    function createDefaultProjects() {
        const today = new Date();
        const tomorrow = new Date(today);
        tomorrow.setDate(today.getDate() + 1);
        tomorrow.setHours(9, 0, 0, 0);
        const tomEnd = new Date(tomorrow);
        tomEnd.setHours(17, 0, 0, 0);

        const dayAfter = new Date(today);
        dayAfter.setDate(today.getDate() + 2);
        dayAfter.setHours(13, 0, 0, 0);
        const afterEnd = new Date(dayAfter);
        afterEnd.setHours(21, 0, 0, 0);

        projects = [{
            id: 'default_' + Date.now(),
            name: 'Основной проект',
            shifts: [
                {
                    id: 'shift1',
                    title: 'Анна Козлова',
                    start: tomorrow.toISOString(),
                    end: tomEnd.toISOString(),
                    backgroundColor: '#f9e79f',
                    borderColor: '#f1c40f',
                    textColor: '#2c3e50',
                    employee: 'Анна Козлова'
                },
                {
                    id: 'shift2',
                    title: 'Борис Лебедев',
                    start: dayAfter.toISOString(),
                    end: afterEnd.toISOString(),
                    backgroundColor: '#aed6f1',
                    borderColor: '#3498db',
                    textColor: '#2c3e50',
                    employee: 'Борис Лебедев'
                }
            ]
        }];
        saveToLocal();
    }

    // цвет для новых смен
    function getColorForEmployee(name) {
        const colors = ['#f9e79f', '#aed6f1', '#f5cba7', '#d5f5e3', '#fadbd8', '#e8daef'];
        let hash = 0;
        for (let i = 0; i < name.length; i++) hash += name.charCodeAt(i);
        return colors[hash % colors.length];
    }

    // ----------------------------------------------
    // 2. FULLCALENDAR И ОБНОВЛЕНИЕ
    // ----------------------------------------------
    let calendar;
    function refreshCalendar() {
        const currentProject = projects.find(p => p.id === activeProjectId);
        if (calendar) {
            calendar.removeAllEvents();
            if (currentProject && currentProject.shifts) {
                calendar.addEventSource(currentProject.shifts);
            }
        }
        // обновить выпадающий список
        const select = document.getElementById('projectSelect');
        select.innerHTML = '';
        projects.forEach(proj => {
            const opt = document.createElement('option');
            opt.value = proj.id;
            opt.textContent = proj.name;
            if (proj.id === activeProjectId) opt.selected = true;
            select.appendChild(opt);
        });
        select.disabled = false;
        document.getElementById('deleteProjectBtn').disabled = (projects.length <= 1);
    }

    // ----------------------------------------------
    // 3. ПРОЕКТЫ
    // ----------------------------------------------
    function createNewProject() {
        const name = prompt('Введите название проекта', 'Новая локация');
        if (!name) return;
        const newId = 'proj_' + Date.now() + '_' + Math.random().toString(36).substr(2, 6);
        projects.push({
            id: newId,
            name: name,
            shifts: []
        });
        activeProjectId = newId;
        saveToLocal();
        refreshCalendar();
    }

    function deleteCurrentProject() {
        if (projects.length <= 1) {
            alert('Нельзя удалить последний проект');
            return;
        }
        const idx = projects.findIndex(p => p.id === activeProjectId);
        if (idx !== -1) {
            projects.splice(idx, 1);
            activeProjectId = projects[0].id;
            saveToLocal();
            refreshCalendar();
        }
    }

    // ----------------------------------------------
    // 4. СМЕНЫ + МОДАЛКА
    // ----------------------------------------------
    const modal = document.getElementById('shiftModal');
    let selectedStart = null, selectedEnd = null;

    function openShiftModal(start, end) {
        selectedStart = new Date(start);
        selectedEnd = new Date(end);
        const formatLocal = (d) => {
            const year = d.getFullYear();
            const month = String(d.getMonth() + 1).padStart(2, '0');
            const day = String(d.getDate()).padStart(2, '0');
            const hours = String(d.getHours()).padStart(2, '0');
            const minutes = String(d.getMinutes()).padStart(2, '0');
            return `${year}-${month}-${day}T${hours}:${minutes}`;
        };
        document.getElementById('startInput').value = formatLocal(selectedStart);
        document.getElementById('endInput').value = formatLocal(selectedEnd);
        modal.style.display = 'flex';
    }

    function closeModal() {
        modal.style.display = 'none';
    }

    function addShiftFromModal() {
        const employee = document.getElementById('employeeSelect').value;
        const startVal = document.getElementById('startInput').value;
        const endVal = document.getElementById('endInput').value;
        if (!startVal || !endVal) {
            alert('Укажите дату и время');
            return;
        }
        if (new Date(startVal) >= new Date(endVal)) {
            alert('Начало не может быть позже окончания');
            return;
        }

        const project = projects.find(p => p.id === activeProjectId);
        if (!project) return;

        const newEvent = {
            id: Date.now() + '_' + Math.random().toString(36).substr(2, 8),
            title: employee,
            start: startVal,
            end: endVal,
            backgroundColor: getColorForEmployee(employee),
            borderColor: '#2c7da0',
            textColor: '#1e293b',
            employee: employee
        };
        project.shifts.push(newEvent);
        saveToLocal();
        refreshCalendar();
        closeModal();
    }

    function deleteShift(eventId) {
        if (!confirm('Удалить эту смену?')) return;
        const project = projects.find(p => p.id === activeProjectId);
        if (project) {
            const idx = project.shifts.findIndex(s => s.id === eventId);
            if (idx !== -1) {
                project.shifts.splice(idx, 1);
                saveToLocal();
                refreshCalendar();
            }
        }
    }

    // ----------------------------------------------
    // 5. ЭКСПОРТ / ИМПОРТ (перенос между устройствами)
    // ----------------------------------------------
    function exportData() {
        const dataStr = JSON.stringify(projects, null, 2);
        const blob = new Blob([dataStr], {type: 'application/json'});
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `shifts_backup_${new Date().toISOString().slice(0,19)}.json`;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
    }

    function importData(file) {
        const reader = new FileReader();
        reader.onload = (e) => {
            try {
                const imported = JSON.parse(e.target.result);
                if (Array.isArray(imported) && imported.length && imported[0].id && Array.isArray(imported[0].shifts)) {
                    projects = imported;
                    activeProjectId = projects[0].id;
                    saveToLocal();
                    refreshCalendar();
                    alert('Данные успешно импортированы!');
                } else {
                    throw new Error();
                }
            } catch(err) {
                alert('Неверный файл. Используйте файл, созданный кнопкой "Экспорт"');
            }
        };
        reader.readAsText(file);
    }

    // ----------------------------------------------
    // 6. ИНИЦИАЛИЗАЦИЯ
    // ----------------------------------------------
    document.addEventListener('DOMContentLoaded', () => {
        // календарь
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
            select: (info) => {
                openShiftModal(info.start, info.end);
            },
            eventClick: (info) => {
                deleteShift(info.event.id);
            },
            height: 'auto',
            weekends: true,
            buttonText: {
                today: 'Сегодня',
                month: 'Месяц',
                week: 'Неделя',
                day: 'День'
            }
        });
        calendar.render();

        // загружаем данные
        loadFromLocal();
        refreshCalendar();

        // обработчики UI
        document.getElementById('newProjectBtn').onclick = createNewProject;
        document.getElementById('deleteProjectBtn').onclick = deleteCurrentProject;
        document.getElementById('exportBtn').onclick = exportData;
        const importInput = document.createElement('input');
        importInput.type = 'file';
        importInput.accept = 'application/json';
        importInput.style.display = 'none';
        document.body.appendChild(importInput);
        importInput.onchange = (e) => {
            if (e.target.files.length) importData(e.target.files[0]);
            importInput.value = '';
        };
        document.getElementById('importBtn').onclick = () => importInput.click();

        document.getElementById('closeModalBtn').onclick = closeModal;
        document.getElementById('saveShiftBtn').onclick = addShiftFromModal;
        document.getElementById('projectSelect').onchange = (e) => {
            activeProjectId = e.target.value;
            refreshCalendar();
        };
        // закрытие модалки по клику вне
        window.onclick = (e) => {
            if (e.target === modal) closeModal();
        };
    });
</script>
</body>
</html>
