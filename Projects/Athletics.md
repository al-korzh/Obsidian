---
area: "[[Areas/Athletics|Athletics]]"
tags:
  - project
lvl: -1
processed: false
startDate:
endDate:
---

> **Цель:** Создать прочную атлетическую базу, трансформировав тело и подготовив его к будущим высоким нагрузкам.

---

### Ключевые результаты

##### Композиция тела:
- [ ] Масса **≤ 85 кг** (с 92 кг).
- [ ] Процент жира **≤ 15%** (с 27%).

##### :
- [ ] **Жим лежа:** **100 кг** на 5-6 повторений.
- [ ] **Подтягивания:** **+10 кг** дополнительного веса на 8-10 повторений.

##### Выносливость:
- [ ] **Контрольный заплыв:** Проплывать **1500 метров** без остановок.

### Активные задачи

```dataview
TABLE status as "Статус", due as "Срок"
FROM #task AND !"Templates"
WHERE project = this.file.link AND status != "done"
SORT due ASC
```

```dataviewjs

// 1. УКАЖИТЕ ТОЧНЫЙ ПУТЬ К ПАПКЕ С ЖУРНАЛАМИ ТРЕНИРОВОК
const FOLDER_PATH = "Projects/Athletics.-1/Logs";

// 2. Получаем страницы, у которых есть свойство (массив) 'type'
const pages = dv.pages(`"${FOLDER_PATH}"`).where(p => p.type);

if (pages.length === 0) {
    dv.paragraph("⚠️ **Данные для таблицы не найдены.** Проверьте, что в файлах-тренировках есть строки с полем `type::`.");
} else {
    // 3. "Разворачиваем" данные: создаем по одной строке на каждое упражнение в каждой заметке
    const exercises = pages.flatMap(p => {
        // Проверяем, что 'type' является массивом
        if (!Array.isArray(p.type)) return [];

        // Для каждого упражнения в массиве 'type' создаем отдельный объект
        return p.type.map((typeName, index) => {
            return {
                exercise: typeName,
                weight: Array.isArray(p.weight) ? p.weight[index] : p.weight,
                reps: Array.isArray(p.reps) ? p.reps[index] : p.reps,
                date: p.date,
                link: p.file.link
            };
        });
    });

    // 4. Группируем полученные объекты по названию упражнения
    const grouped = exercises.groupBy(ex => ex.exercise);

    // 5. Строим таблицу
    dv.table(
        ["Упражнение", "Записей", "Рекордный вес (кг)", "Последний результат", "Дата последней"],
        grouped.map(group => {
            const sortedRows = group.rows.sort(r => r.date, 'desc');
            const latest = sortedRows[0];
            const recordWeight = Math.max(...group.rows.map(r => r.weight || 0));

            // Обработка случая, когда reps может быть массивом
            let repsValue = latest.reps;
            if (Array.isArray(repsValue)) {
                repsValue = repsValue.join(', '); // Если в reps массив, показываем все значения
            }

            return [
                group.key,
                group.rows.length,
                recordWeight,
                `${latest.weight || '?'} x ${repsValue || '?'}`,
                latest.date ? latest.date.toFormat("yyyy-MM-dd") : "Нет даты"
            ];
        })
    );
}
```