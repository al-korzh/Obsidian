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
// --- ФИНАЛЬНАЯ НАДЕЖНАЯ ВЕРСИЯ ---

// 1. Укажите точный путь к папке
const FOLDER_PATH = "Projects/Athletics.-1/Logs";
const pages = dv.pages(`"${FOLDER_PATH}"`);

if (pages.length === 0) {
    dv.paragraph("ℹ️ В папке для логов пока нет ни одного файла.");
} else {
    const exercises = pages
        .flatMap(p => p.file.lists)
        .where(l => l.type);

    if (exercises.length === 0) {
        dv.warn("ПРЕДУПРЕЖДЕНИЕ: Файлы в папке есть, но в них не найдено ни одной строки с полем `type::`.");
    } else {
        const groupedExercises = exercises.groupBy(l => l.type);

        dv.table(
            ["Упражнение", "Записей", "Рекордный вес (кг)", "Последний результат", "Дата последней"],
            groupedExercises.map(group => {
                // САМОЕ ВАЖНОЕ ИЗМЕНЕНИЕ:
                // Сначала отбираем только те записи, где ТОЧНО есть и файл, и дата
                const validRows = group.rows.filter(r => r.file && r.file.date);

                // Если для этого упражнения ВООБЩЕ нет записей с валидной датой
                if (validRows.length === 0) {
                    return [group.key, group.rows.length, "---", "Нет данных с валидной датой", "---"];
                }

                // Сортируем только валидные записи
                const sortedRows = validRows.sort(r => r.file.date, 'desc');
                const latest = sortedRows[0];
                const recordWeight = Math.max(...group.rows.map(r => r.weight || 0));

                return [
                    group.key,
                    group.rows.length,
                    recordWeight,
                    `${latest.weight || '?'} x ${latest.reps || '?'}`,
                    latest.file.date.toFormat("yyyy-MM-dd")
                ];
            })
        );
    }
}
```


```dataviewjs
// --- Финальный диагностический скрипт ---
dv.header(3, "Финальный отчет о свойстве 'date' в файлах-тренировках");

const FOLDER_PATH = "Projects/Athletics.-1/Logs";
const pages = dv.pages(`"${FOLDER_PATH}"`);

if (pages.length === 0) {
    dv.error("ОШИБКА: Не найдено ни одного файла в папке. Проверьте путь.");
} else {
    dv.table(
        ["Файл", "Свойство 'date' существует и читается?"],
        pages.map(p => {
            const dateExists = p.file.date && typeof(p.file.date) === 'object';
            return [
                p.file.link,
                dateExists ? "✅ Да" : "❌ Нет"
            ];
        })
    );
}
```