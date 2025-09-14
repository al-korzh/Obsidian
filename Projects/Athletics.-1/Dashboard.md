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
const FOLDER_PATH = "Projects/Athletics.-1/Logs";
const REQUIRED_TAG = "#gym";

const pages = dv.pages(`"${FOLDER_PATH}" AND ${REQUIRED_TAG}`);

const allExercises = pages.flatMap(page => {
    if (!page.date) return [];
    const exercises = page.file.lists
        .where(item => item.type && item.weight && item.reps && item.sets);
    return exercises.map(ex => ({
        date: page.date.toFormat("yyyy-MM-dd"),
        name: ex.type,
        result: `${ex.weight} x ${ex.reps} x ${ex.sets}`
    }));
});

const exerciseData = {};
const allDates = new Set();

allExercises.forEach(ex => {
    if (!exerciseData[ex.name]) {
        exerciseData[ex.name] = {};
    }
    exerciseData[ex.name][ex.date] = ex.result;
    allDates.add(ex.date);
});

const sortedDates = Array.from(allDates).sort();
const headers = ["Упражнение", ...sortedDates];

const rows = Object.keys(exerciseData).sort().map(exerciseName => {
    const row = [exerciseName];
    sortedDates.forEach(date => {
        row.push(exerciseData[exerciseName][date] || "—");
    });
    return row;
});

if (rows.length > 0) {
    dv.table(headers, rows);
} else {
    dv.paragraph("💪 Не найдено данных с тегом #gym в указанной папке.");
}
```


```dataviewjs
// --- НАСТРОЙКИ ---
const FOLDER_PATH = "Projects/Athletics.-1/Logs";
const REQUIRED_TAG = "#gym";
// --- КОНЕЦ НАСТРОЕК ---

// 1. Находим все страницы
const pages = dv.pages(`"${FOLDER_PATH}" AND ${REQUIRED_TAG}`);

// 2. Собираем все упражнения в один массив
const workoutData = pages.flatMap(page => {
    // Безопасная проверка: если у страницы нет даты, пропускаем ее
    if (!page.date) {
        return []; // Возвращаем пустой массив, чтобы ничего не добавилось
    }

    // Находим все строки-упражнения
    const exercises = page.file.lists
        .where(item => item.type && item.weight && item.reps && item.sets);
        
    // Для каждого упражнения создаем одну строку в будущей таблице
    return exercises.map(ex => [
        page.date.toFormat("yyyy-MM-dd"), // Дата
        ex.type,                          // Упражнение
        ex.weight,                        // Вес
        ex.reps,                          // Повторения
        ex.sets                           // Подходы
    ]);
});

// 3. Сортируем все записи по дате (от новой к старой)
workoutData.sort((a, b) => b[0].localeCompare(a[0]));

// 4. Выводим таблицу
if (workoutData.length > 0) {
    dv.table(
        ["Дата", "Упражнение", "Вес (кг)", "Повторения", "Подходы"],
        workoutData
    );
} else {
    dv.paragraph("💪 Не найдено данных с тегом #gym в указанной папке.");
}
