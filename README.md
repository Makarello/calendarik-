# calendarik-

<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Календарь смен – несколько проектов</title>
    <!-- FullCalendar CSS -->
    <link href="https://cdn.jsdelivr.net/npm/fullcalendar@5.11.3/main.min.css" rel="stylesheet">
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
            background-color: #f5f5f5;
        }
        #calendar {
            max-width: 1100px;
            margin: 20px auto 0;
            background: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        /* Панель управления проектами */
        .project-bar {
            max-width: 1100px;
            margin: 0 auto 15px;
            display: flex;
            gap: 10px;
            align-items: center;
            background: white;
            padding: 15px 20px;
            border-radius: 8px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
        }
        .project-bar select {
            flex: 1;
            padding: 8px 12px;
            font-size: 16px;
            border: 1px solid #ccc;
            border-radius: 4px;
        }
        .project-bar button {
            padding: 8px 16px;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            font-size: 14px;
            background-color: #4CAF50;
            color: white;
        }
        .project-bar button.delete {
            background-color: #f44336;
        }
        .project-bar button:hover {
            opacity: 0.9;
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
            border-radius: 8px;
            width: 350px;
            box-shadow: 0 4px 20px rgba(0,0,0,0.2);
        }
        .modal-content h3 {
            margin-top: 0;
            color: #333;
        }
        .form-group {
            margin-bottom: 15px;
        }
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: bold;
            color: #555;
        }
        .form-group input, .form-group select {
            width: 100%;
            padding: 8px;
            border: 1px solid #ddd;
            border-radius: 4px;
            box-sizing: border-box;
        }
        .button-group {
            display: flex;
            justify-content: flex-end;
            gap: 10px;
            margin-top: 20px;
        }
        .btn-primary {
            background-color: #4CAF50;
            color: white;
        }
        .btn-cancel {
            background-color: #ccc;
            color: #333;
        }
    </style>
</head>
<body>
    <!-- Панель проектов -->
    <div class="project-bar">
        <select id="projectSelect"></select>
        <button id="newProjectBtn">➕ Новый проект</button>
        <button id="deleteProjectBtn" class="delete">🗑️ Удалить</button>
    </div>

    <div id="calendar"></div>

    <!-- Модальное окно для добавления смены -->
    <div id="shiftModal" class="modal">
        <div class="modal-content">
            <h3 id="modalTitle">Добавить смену</h3>
            <div class="form-group">
                <label for="employeeSelect">Сотрудник</label>
                <select id="employeeSelect">
                    <option value="Иван Петров">Иван Петров</option>
                    <option value="Мария Смирнова">Мария Смирнова</option>
                    <option value="Алексей Иванов">Алексей Иванов</option>
                    <option value="Елена Кузнецова">Елена Кузнецова</option>
                    <option value="Дмитрий Соколов">Дмитрий Соколов</option>
                </select>
            </div>
            <div class="form-group">
                <label for="startTime">Начало</label>
                <input type="datetime-local" id="startTime" required>
            </div>
            <div class="form-group">
                <label for="endTime">Конец</label>
                <input type="datetime-local" id="endTime" required>
            </div>
            <div class="button-group">
                <button type="button" class="btn-cancel" onclick="closeModal()">Отмена</button>
                <button type="button" class="btn-primary" onclick="saveShift()">Сохранить</button>
            </div>
        </div>
    </div>

    <!-- FullCalendar JS -->
    <script src="https://cdn.jsdelivr.net/npm/fullcalendar@5.11.3/main.min.js"></script>
    <script>
        // ---------- Глобальные переменные ----------
        let calendar;
        let projects = [];              // массив проектов: [{ id, name, shifts }]
        let activeProjectId = null;      // id текущего проекта

        // Элементы управления проектами
        const projectSelect = document.getElementById('projectSelect');
        const newProjectBtn = document.getElementById('newProjectBtn');
        const deleteProjectBtn = document.getElementById('deleteProjectBtn');

        // Элементы модального окна
        const modal = document.getElementById('shiftModal');
        const employeeSelect = document.getElementById('employeeSelect');
        const startTimeInput = document.getElementById('startTime');
        const endTimeInput = document.getElementById('endTime');
        const modalTitle = document.getElementById('modalTitle');

        // Переменные для выделенного интервала
        let selectedStart = null;
        let selectedEnd = null;

        // ---------- Загрузка и сохранение в localStorage ----------
        const STORAGE_KEY = 'shiftCalendarProjects';

        function loadProjects() {
            const saved = localStorage.getItem(STORAGE_KEY);
            if (saved) {
                try {
                    projects = JSON.parse(saved);
                    // Убедимся, что у каждого проекта есть поле shifts (массив)
                    projects = projects.filter(p => p && p.id && p.name && Array.isArray(p.shifts));
                } catch (e) {
                    console.warn('Ошибка загрузки, создаём проект по умолчанию');
                    projects = [];
                }
            }
            // Если проектов нет, создаём демонстрационный
            if (projects.length === 0) {
                createDefaultProject();
            }
            // Устанавливаем активный проект (первый, если не задан)
            if (!activeProjectId || !projects.find(p => p.id === activeProjectId)) {
                activeProjectId = projects[0].id;
            }
        }

        function saveProjects() {
            localStorage.setItem(STORAGE_KEY, JSON.stringify(projects));
        }

        // Создание проекта по умолчанию с тестовыми сменами
        function createDefaultProject() {
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

            const demoShifts = [
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
            ];

            projects.push({
                id: 'proj_' + Date.now(),
                name: 'Основной проект',
                shifts: demoShifts
            });
        }

        // Генерация цвета на основе имени
        function getRandomColor(name) {
            let hash = 0;
            for (let i = 0; i < name.length; i++) {
                hash = name.charCodeAt(i) + ((hash << 5) - hash);
            }
            const hue = hash % 360;
            return `hsl(${hue}, 70%, 85%)`;
        }

        // ---------- Работа с проектами (выпадающий список) ----------
        function renderProjectSelect() {
            projectSelect.innerHTML = '';
            projects.forEach(p => {
                const option = document.createElement('option');
                option.value = p.id;
                option.textContent = p.name;
                if (p.id === activeProjectId) option.selected = true;
                projectSelect.appendChild(option);
            });
        }

        // Переключение активного проекта
        function switchProject(projectId) {
            if (!projectId) return;
            const project = projects.find(p => p.id === projectId);
            if (!project) return;
            activeProjectId = projectId;
            // Обновляем календарь событиями этого проекта
            calendar.removeAllEvents();
            calendar.addEventSource(project.shifts);
            renderProjectSelect();
        }

        // Создать новый проект
        function createNewProject() {
            const name = prompt('Введите название нового проекта:', 'Новый проект');
            if (!name) return;
            const newId = 'proj_' + Date.now() + '_' + Math.random().toString(36).substr(2, 5);
            projects.push({
                id: newId,
                name: name,
                shifts: []
            });
            saveProjects();
            activeProjectId = newId;
            switchProject(newId);
        }

        // Удалить текущий проект
        function deleteCurrentProject() {
            if (projects.length <= 1) {
                alert('Нельзя удалить единственный проект. Создайте новый перед удалением.');
                return;
            }
            if (!confirm(`Удалить проект "${projects.find(p => p.id === activeProjectId)?.name}"? Все смены будут потеряны.`)) return;
            const index = projects.findIndex(p => p.id === activeProjectId);
            if (index !== -1) {
                projects.splice(index, 1);
                activeProjectId = projects[0].id; // переключаемся на первый оставшийся
                saveProjects();
                switchProject(activeProjectId);
            }
        }

        // ---------- Модальное окно ----------
        function openModal(start, end) {
            selectedStart = start;
            selectedEnd = end;

            const formatDateForInput = (date) => {
                const d = new Date(date);
                const year = d.getFullYear();
                const month = String(d.getMonth() + 1).padStart(2, '0');
                const day = String(d.getDate()).padStart(2, '0');
                const hours = String(d.getHours()).padStart(2, '0');
                const minutes = String(d.getMinutes()).padStart(2, '0');
                return `${year}-${month}-${day}T${hours}:${minutes}`;
            };

            startTimeInput.value = formatDateForInput(start);
            endTimeInput.value = formatDateForInput(end);

            modalTitle.innerText = 'Добавить смену';
            modal.style.display = 'flex';
        }

        window.closeModal = function() {
            modal.style.display = 'none';
        };

        // Сохранить смену (добавляет в текущий проект)
        window.saveShift = function() {
            const employee = employeeSelect.value;
            const start = startTimeInput.value;
            const end = endTimeInput.value;

            if (!employee || !start || !end) {
                alert('Заполните все поля');
                return;
            }

            if (new Date(start) >= new Date(end)) {
                alert('Время начала должно быть раньше времени окончания');
                return;
            }

            const project = projects.find(p => p.id === activeProjectId);
            if (!project) return;

            const newEvent = {
                id: String(Date.now()) + '_' + Math.random().toString(36).substr(2, 5),
                title: employee,
                start: start,
                end: end,
                backgroundColor: getRandomColor(employee),
                borderColor: '#3788d8',
                textColor: '#000',
                employee: employee
            };

            project.shifts.push(newEvent);
            saveProjects();

            // Обновляем календарь
            calendar.addEvent(newEvent);
            closeModal();
        };

        // Удалить смену (из текущего проекта)
        window.deleteShift = function(eventId) {
            if (!confirm('Удалить эту смену?')) return;
            const project = projects.find(p => p.id === activeProjectId);
            if (!project) return;
            const index = project.shifts.findIndex(s => s.id === eventId);
            if (index !== -1) {
                project.shifts.splice(index, 1);
                saveProjects();
                // Удаляем событие из календаря
                const event = calendar.getEventById(eventId);
                if (event) event.remove();
            }
        };

        // ---------- Инициализация календаря ----------
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
                }
            });

            calendar.render();

            // Загружаем проекты из localStorage
            loadProjects();

            // Заполняем выпадающий список и показываем активный проект
            renderProjectSelect();
            switchProject(activeProjectId);

            // Обработчики кнопок проектов
            newProjectBtn.addEventListener('click', () => {
                createNewProject();
            });

            deleteProjectBtn.addEventListener('click', () => {
                deleteCurrentProject();
            });

            projectSelect.addEventListener('change', (e) => {
                switchProject(e.target.value);
            });
        });
    </script>
</body>
</html>
