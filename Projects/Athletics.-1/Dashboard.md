---
tags:
  - project
areas: "[[Areas/Athletics|Athletics]]"
---
> **Цель:** Создать прочную атлетическую базу, трансформировав тело и подготовив его к будущим высоким нагрузкам.

- **[[Kanban|Доска задач]]**
- **Аналитика:** [[1. Projects/НАЗВАНИЕ ПАПКИ ПРОЕКТА/Аналитика/]]
- **План тренировок:** [[1. Projects/НАЗВАНИЕ ПАПКИ ПРОЕКТА/План тренировок - Этап 1]] 
- **Журнал тренировок:** [[1. Projects/НАЗВАНИЕ ПАПКИ ПРОЕКТА/Журнал тренировок/]] 
---

### KPI

##### Тело (композиция):
- [ ] **Процент жира:** Снизить до **≤ 18%** (с 27%).
- [ ] **Вес тела:** Достичь **~83-85 кг** (с 92 кг).
- [ ] **Обхват талии:** Уменьшить на **8-12 см**.

##### Сила (стабильные рабочие веса):
- [ ] **Жим лежа:** **100 кг** на 5-6 повторений.
- [ ] **Подтягивания:** **+10 кг** дополнительного веса на 8-10 повторений.
- [ ] **Отжимания на брусьях:** **+15-20 кг** дополнительного веса на 8-10 повторений.
- [ ] **Жим ногами:** **200-220 кг** на 8-10 повторений.

##### Выносливость (плавание):
- [ ] **Контрольный заплыв:** Проплывать **1500 метров** без остановок.
- [ ] **Крейсерская скорость:** Выполнять серию **10х100м**, удерживая темп **1:40-1:45** на каждом отрезке.
- [ ] **Средний объем тренировки:** **2.5 - 3 км**.


### Задачи

```dataview
TABLE status as "Статус", due as "Срок" FROM #task AND !"Templates" WHERE project = this.file.link AND status != "done" SORT due ASC
```


### Прошедшие тренировки


```dataviewjs
// --- НАСТРОЙКИ ---
const FOLDER_PATH = "Projects/Athletics.-1/Logs";
const REQUIRED_TAG = "#gym";
// --- КОНЕЦ НАСТРОЕК ---

// 1. Находим все страницы в папке с нужным тегом
const pages = dv.pages(`"${FOLDER_PATH}" AND ${REQUIRED_TAG}`);

// 2. Собираем все упражнения в один массив, сохраняя все детали
const allExercises = pages.flatMap(page => {
    if (!page.date) return [];

    const exercises = page.file.lists
        .where(item => item.type && item.weight && item.reps && item.sets);
        
    return exercises.map(ex => ({
        date: page.date.toFormat("yyyy-MM-dd"),
        name: ex.type,
        details: { // Сохраняем все поля как объект
            weight: ex.weight,
            reps: ex.reps,
            sets: ex.sets
        }
    }));
});

// 3. Группируем данные по названию упражнения
const exerciseData = {};
const allDates = new Set(); 

allExercises.forEach(ex => {
    if (!exerciseData[ex.name]) {
        exerciseData[ex.name] = {};
    }
    // В ячейку [Упражнение][Дата] кладем объект с деталями
    exerciseData[ex.name][ex.date] = ex.details;
    allDates.add(ex.date);
});

// 4. Готовим заголовки и данные для финальной таблицы
const sortedDates = Array.from(allDates).sort();

// Создаем мульти-колонки для заголовков
const headers = ["Упражнение"];
sortedDates.forEach(date => {
    // Для каждой даты добавляем три заголовка
    headers.push(`${date} (Вес)`);
    headers.push(`${date} (Повт)`);
    headers.push(`${date} (Подх)`);
});

// Создаем строки для таблицы
const rows = Object.keys(exerciseData).sort().map(exerciseName => {
    const row = [exerciseName]; 
    // Проходим по всем датам в том же порядке, что и в заголовках
    sortedDates.forEach(date => {
        const data = exerciseData[exerciseName][date];
        if (data) {
            // Если для этой даты есть результат - добавляем все три значения
            row.push(data.weight);
            row.push(data.reps);
            row.push(data.sets);
        } else {
            // Если нет - ставим три прочерка
            row.push("—");
            row.push("—");
            row.push("—");
        }
    });
    return row;
});

// 5. Выводим таблицу
if (rows.length > 0) {
    dv.table(headers, rows);
} else {
    dv.paragraph("💪 Не найдено данных с тегом #gym в указанной папке.");
}
```
