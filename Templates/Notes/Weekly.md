---
tags:
  - inbox
  - weekly
date:
processed: false
---
<%* 
// 1. Указываем путь к папке с шаблонами обзоров 
const folderPath = "Templates/Area Reviews"; 

// 2. Получаем все файлы из этой папки 
const reviewFiles = app.vault.getFiles().filter(file => file.path.startsWith(folderPath));

// 3. Сортируем файлы по имени (необязательно, но делает порядок предсказуемым)
reviewFiles.sort((a, b) => a.name.localeCompare(b.name));
// 4. Вставляем содержимое каждого файла в цикле
for (const file of reviewFiles) { 
	await tp.file.include(`[[${file.path}]]`); 
	tR += "\n\n---\n\n"; // Добавляем разделитель для красоты
} 
%>