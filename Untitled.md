# Тест парсинга

- Жим лежа (exercise:: "Жим лежа"; weight:: 80, reps:: 8, sets:: 4)

```dataview
TABLE
    item.exercise, 
    item.weight, 
    item.reps, 
    item.sets
FROM ""
FLATTEN file.lists as item
WHERE item.exercise
```
