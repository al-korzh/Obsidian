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
new Notice(`1. Ищу файлы в папке: "${folderPath}"`);

// 2. Получаем все файлы из этой папки 
const reviewFiles = app.vault.getFiles().filter(file => file.path.startsWith(folderPath));

new Notice(`2. Найдено подходящих файлов: ${reviewFiles.length}`);

if (reviewFiles.length === 0) { 
	new Notice("ОШИБКА: Файлы не найдены. Проверьте, что путь в скрипте и название папки совпадают на 100%.", 10000); 
}

// 3. Сортируем файлы по имени (необязательно, но делает порядок предсказуемым)
reviewFiles.sort((a, b) => a.name.localeCompare(b.name));
// 4. Вставляем содержимое каждого файла в цикле
for (const file of reviewFiles) { 
	const areaName = file.basename;
	const header = `### [[${areaName}]]\n\n`;
	tR += header;
	tR += await tp.file.include(`[[${file.path}]]`); 
	// Добавляем разделитель для красоты
	tR += "\n\n---\n\n"; 
} 
%>