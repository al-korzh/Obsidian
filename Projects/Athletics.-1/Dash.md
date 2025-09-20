---
tags:
  - project
status: active
area: "[[Areas/Athletics|Атлетика]]"
startDate: 2025-09-20
endDate: # Укажите примерную дату завершения
---

> **Цель:** Создать прочную атлетическую базу, трансформировав тело и подготовив его к будущим высоким нагрузкам.

---
### 🕹️ Панель управления
- **Доска задач:** [[Активные задачи (Kanban)]]
- **План тренировок:** [[1. Projects/НАЗВАНИЕ ПАПКИ ПРОЕКТА/План тренировок - Этап 1]]
- **Аналитика прогрессии:** [[1. Projects/НАЗВАНИЕ ПАПКИ ПРОЕКТА/Аналитика/Прогрессия весов]]
- **Журнал тренировок (папка):** [[1. Projects/НАЗВАНИЕ ПАПКИ ПРОЕКТА/Журнал тренировок/]]

---
### 🎯 Ключевые результаты (KPI)

##### Тело (композиция):
- [ ] **Процент жира:** Снизить до **≤ 18%** (с 27%).
- [ ] **Вес тела:** Достичь **~83-85 кг** (с 92 кг).

##### Сила (стабильные рабочие веса):
- [ ] **Жим лежа:** **100 кг** на 5-6 повторений.
- [ ] **Подтягивания:** **+10 кг** дополнительного веса на 8-10 повторений.

##### Выносливость (плавание):
- [ ] **Контрольный заплыв:** Проплывать **1500 метров** без остановок.

### ✅ Активные задачи
> *Автоматически собирает все невыполненные задачи по этому проекту.*

```dataview
TABLE status as "Статус", due as "Срок"
FROM #task AND !"Templates"
WHERE project = this.file.link AND status != "done"
SORT due ASC
```

```dataviewjs
// --- ФИНАЛЬНАЯ НАДЕЖНАЯ ВЕРСИЯ ---
dv.header(3, "Таблица прогрессии");

// 1. УКАЖИТЕ ТОЧНЫЙ ПУТЬ К ПАПКЕ С ЖУРНАЛАМИ ТРЕНИРОВОК
const FOLDER_PATH = "Projects/Athletics.-1/Logs"; // <--- ЗАМЕНИТЕ НА ВАШ ПУТЬ

const pages = dv.pages(`"${FOLDER_PATH}"`);

if (pages.length === 0) {
    dv.paragraph("ℹ️ В папке для логов пока нет ни одного файла.");
} else {
    const exercises = pages
        .flatMap(p => p.file.lists)
        .where(l => l.type && l.file && l.file.date);

    if (exercises.length === 0) {
        dv.paragraph("⚠️ **Данные для таблицы не найдены.** Проверьте, что в файлах-тренировках есть свойство 'date' и строки с полем `type::`.");
    } else {
        const groupedExercises = exercises.groupBy(l => l.type);

        dv.table(
            ["Упражнение", "Записей", "Рекордный вес (кг)", "Последний результат", "Дата последней"],
            groupedExercises.map(group => {
                // Защита от отсутствия даты: сначала отбираем только те записи, где ТОЧНО есть и файл, и дата
                const validRows = group.rows.filter(r => r.file && r.file.date);

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
