# calendarik-

<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Календарь смен — с Supabase</title>
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
         .loader {
            text-align: center;
            padding: 20px;
            font-size: 1.2em;
            color: #555;
        }
    </style>
    <!-- Подключаем Supabase JS SDK -->
    <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
</head>
<body>
    <div class="project-bar">
        <select id="projectSelect" disabled>
            <option>Загрузка проектов...</option>
        </select>
        <button id="newProjectBtn" disabled>➕ Новый проект</button>
        <button id="deleteProjectBtn" class="delete" disabled>🗑️ Удалить</button>
    </div>
    <div id="calendar"></div>

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

    <script src="https://cdn.jsdelivr.net/npm/fullcalendar@5.11.3/main.min.js"></script>
    <script>
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
        // 1. СЮДА ВСТАВЬТЕ ВАШИ ДАННЫЕ ИЗ SUPABASE
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
        const SUPABASE_URL = 'sb_publishable_QraoGOwTBh2cB0NRESgaeg_RwBqQhT7';
        const SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImlveWJ2anV0ZGZid2N5emttdnNtIiwicm9sZSI6InNlcnZpY2Vfcm9sZSIsImlhdCI6MTc3NzA0OTY2NiwiZXhwIjoyMDkyNjI1NjY2fQ.O8oUbYGzZ7noT0VD33sQYV9D8X2n_axh_K60321QsRE'; // 
        // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

        // Инициализация клиента Supabase
        const supabase = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

        // Глобальные переменные
        let calendar;
        let projects = [];      // массив проектов из БД
        let activeProjectId = null;

        // Элементы DOM
        const projectSelect = document.getElementById('projectSelect');
        const newProjectBtn = document.getElementById('newProjectBtn');
        const deleteProjectBtn = document.getElementById('deleteProjectBtn');
        const modal = document.getElementById('shiftModal');
        const employeeSelect = document.getElementById('employeeSelect');
        const startTimeInput = document.getElementById('startTime');
        const endTimeInput = document.getElementById('endTime');
        const modalTitle = document.getElementById('modalTitle');

        let selectedStart = null, selectedEnd = null;

        // --- Вспомогательные функции ---
        function getRandomColor(name) {
            let hash = 0;
            for (let i = 0; i < name.length; i++) {
                hash = name.charCodeAt(i) + ((hash << 5) - hash);
            }
            const hue = hash % 360;
            return `hsl(${hue}, 70%, 85%)`;
        }

        // --- Функции загрузки/сохранения в Supabase ---
        async function loadProjectsFromSupabase() {
            let { data, error } = await supabase
                .from('projects')
                .select('*');
            
            if (error) {
                console.error('Ошибка загрузки проектов:', error);
                alert('Не удалось загрузить данные из Supabase. Проверьте настройки.');
                return;
            }

            if (data && data.length > 0) {
                projects = data;
            } else {
                // Если в базе пусто, создадим проект по умолчанию
                console.log("База данных пуста, создаю проект по умолчанию...");
                await createDefaultProject();
                // После создания по умолчанию, перезагружаем страницу, чтобы отобразить новый проект
                // Но можно просто вызвать loadProjectsFromSupabase() ещё раз,
                // но здесь для простоты предложим пользователю обновить страницу или делаем рекурсию.
                // Мы просто создадим проект, а loadProjectsFromSupabase() вызовется в конце init.
            }
            // Обновляем интерфейс
            if (projects.length > 0 && !activeProjectId) {
                activeProjectId = projects[0].id;
            }
            renderUI();
        }

        async function createDefaultProject() {
            const defaultProject = {
                id: 'proj_default_' + Date.now(),
                name: 'Основной проект',
                shifts: [] // Начнём с пустым проектом, без демо-смен
            };
            let { error } = await supabase
                .from('projects')
                .insert([defaultProject]);
            
            if (error) {
                console.error('Ошибка создания проекта по умолчанию:', error);
                alert('Не удалось создать проект по умолчанию.');
            } else {
                projects.push(defaultProject);
                activeProjectId = defaultProject.id;
                renderUI();
            }
        }

        async function syncProjectsToSupabase() {
            // Этот метод принимает текущий массив projects и сохраняет его в Supabase.
            // Так как используется политика "Enable all", проще всего удалить старые и вставить новые.
            let { error: deleteError } = await supabase
                .from('projects')
                .delete()
                .neq('id', 'non-existent-id'); // Удаляем всё
            if (deleteError) {
                console.error('Ошибка синхронизации (удаление):', deleteError);
                alert('Ошибка сохранения данных (удаление)');
                return;
            }
            let { error: insertError } = await supabase
                .from('projects')
                .insert(projects);
            if (insertError) {
                console.error('Ошибка синхронизации (вставка):', insertError);
                alert('Ошибка сохранения данных (вставка)');
                return;
            }
            console.log('Данные успешно синхронизированы с Supabase');
        }

        async function updateProjectInSupabase(project) {
            // Upsert: если проект с таким id существует, обновит его, иначе создаст.
            let { error } = await supabase
                .from('projects')
                .upsert(project, { onConflict: 'id' });
            if (error) {
                console.error('Ошибка обновления проекта в Supabase:', error);
                alert('Не удалось сохранить изменения проекта.');
            }
        }

        // --- Функции для работы с проектами ---
        function renderUI() {
            if (!projects.length) {
                projectSelect.innerHTML = '<option>Нет проектов, создайте новый</option>';
                projectSelect.disabled = true;
                newProjectBtn.disabled = false;
                deleteProjectBtn.disabled = true;
                calendar.removeAllEvents();
                return;
            }
            // Заполняем select
            projectSelect.innerHTML = '';
            projects.forEach(p => {
                const option = document.createElement('option');
                option.value = p.id;
                option.textContent = p.name;
                if (p.id === activeProjectId) option.selected = true;
                projectSelect.appendChild(option);
            });
            projectSelect.disabled = false;
            newProjectBtn.disabled = false;
            deleteProjectBtn.disabled = false;

            // Показываем смены активного проекта
            const activeProject = projects.find(p => p.id === activeProjectId);
            if (activeProject) {
                calendar.removeAllEvents();
                calendar.addEventSource(activeProject.shifts || []);
            } else {
                calendar.removeAllEvents();
            }
        }

        async function createNewProject() {
            const name = prompt('Введите название нового проекта:', 'Новый проект');
            if (!name) return;
            const newId = 'proj_' + Date.now() + '_' + Math.random().toString(36).substr(2, 5);
            const newProject = {
                id: newId,
                name: name,
                shifts: []
            };
            projects.push(newProject);
            activeProjectId = newId;
            renderUI();
            await updateProjectInSupabase(newProject);
        }

        async function deleteCurrentProject() {
            if (projects.length <= 1) {
                alert('Нельзя удалить единственный проект. Создайте новый перед удалением.');
                return;
            }
            const projectName = projects.find(p => p.id === activeProjectId)?.name;
            if (!confirm(`Удалить проект "${projectName}"? Все смены будут потеряны.`)) return;
            const index = projects.findIndex(p => p.id === activeProjectId);
            if (index !== -1) {
                const deletedProject = projects[index];
                projects.splice(index, 1);
                activeProjectId = projects[0].id;
                renderUI();
                let { error } = await supabase
                    .from('projects')
                    .delete()
                    .eq('id', deletedProject.id);
                if (error) {
                    console.error('Ошибка удаления проекта из Supabase:', error);
                    alert('Не удалось удалить проект на сервере.');
                }
            }
        }

        function switchProject(projectId) {
            if (!projectId) return;
            if (!projects.find(p => p.id === projectId)) return;
            activeProjectId = projectId;
            renderUI();
        }

        // --- Операции со сменами ---
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

        async function saveShift() {
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

            const projectIndex = projects.findIndex(p => p.id === activeProjectId);
            if (projectIndex === -1) return;

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
            projects[projectIndex].shifts.push(newEvent);
            renderUI();
            await updateProjectInSupabase(projects[projectIndex]);
            closeModal();
        }

        window.deleteShift = async function(eventId) {
            if (!confirm('Удалить эту смену?')) return;
            const projectIndex = projects.findIndex(p => p.id === activeProjectId);
            if (projectIndex === -1) return;
            const initialLength = projects[projectIndex].shifts.length;
            projects[projectIndex].shifts = projects[projectIndex].shifts.filter(s => s.id !== eventId);
            if (projects[projectIndex].shifts.length === initialLength) return;
            renderUI();
            await updateProjectInSupabase(projects[projectIndex]);
        };

        // --- Инициализация календаря и загрузка данных из Supabase ---
        document.addEventListener('DOMContentLoaded', async function() {
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

            // Загружаем данные из Supabase
            await loadProjectsFromSupabase();

            // Обработчики кнопок
            newProjectBtn.addEventListener('click', createNewProject);
            deleteProjectBtn.addEventListener('click', deleteCurrentProject);
            projectSelect.addEventListener('change', (e) => switchProject(e.target.value));
        });
    </script>
</body>
</html>
