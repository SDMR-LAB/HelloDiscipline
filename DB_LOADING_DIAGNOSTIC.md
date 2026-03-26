# 🔍 Диагностический отчёт: Проблема загрузки данных из БД

## Проблема
У пользователя: "Не отображаются записи из БД в генераторе отчётов" 
- Когда пользователь сохраняет report в БД через "Сохранить в БД" кнопку, данные якобы сохраняются
- Но когда пользователь нажимает "Загрузить из БД" и выбирает дату, select остаётся пустой

## 📊 Цепь обработки данных

### 1. СОХРАНЕНИЕ (saveToDatabase())
```
JavaScript: saveToDatabase()
  ↓
POST /api/completions/create
  ↓
core/api.py: create_item() → db.insert()
  ↓
SQLite: INSERT INTO completions (date, day_number, state, ...) VALUES (...)
  ↓
Затем сохраняются completion_habits
```

### 2. ЗАГРУЗКА (loadDatesFromDB())
```
JavaScript: loadDatesFromDB()
  ↓
GET /api/stats/period?period=all
  ↓
core/stats_api.py: db.list(Completion)
  ↓
SQLite: SELECT * FROM completions
  ↓
Возврат: {status: success, days_data: [...дни...]}
  ↓
JavaScript: заполнить select из days_data[].date
```

## ✅ Проверки что сделаны

1. **Database class** - добавлен метод `execute()` для поддержки migrations
2. **API endpoints** - `/api/completions/list?date=...` работает правильно
3. **Date handling** - Field.to_db() и Field.from_db() преобразуют даты правильно (YYYY-MM-DD)
4. **Entity registration** - Completion регистрируется в EntityMeta._entities при загрузке
5. **Logging** - добавлено подробное логирование на каждом шаге

## 🔧 Добавленные инструменты для диагностики

### Debug API endpoints:
```
POST /api/debug/test-completion
  - Создаёт тестовую completion для сегодняшней даты
  - Используйте для проверки если сохранение работает

GET /api/debug/list-completions
  - Выводит ВСЕ completions в БД
  - Используйте для проверки если completions есть в БД
```

### Enhanced logging в report.js:
- `saveToDatabase()` - 7 шагов с логированием (🔵 STEP 1-7)
- `loadDatesFromDB()` - 5 шагов с логированием  
- `loadDayFromDB()` - 4 шага с логированием
- Все ошибки обозначены как 🔴 ERROR или ⚠️ WARNING

## 🧪 Как тестировать

1. **Запустите приложение:**
   ```bash
   python app.py
   ```

2. **Откройте браузер консоль (F12):**
   - Вкладка "Console"
   - Следите за 🔵 STEP логами

3. **Тест 1: Создание тестовой completion:**
   ```javascript
   fetch('/api/debug/test-completion', {method: 'POST'})
     .then(r => r.json())
     .then(d => console.log('Test result:', d))
   ```

4. **Тест 2: Проверка BД:**
   ```javascript
   fetch('/api/debug/list-completions')
     .then(r => r.json())
     .then(d => console.log('All completions:', d))
   ```

5. **Тест 3: Загрузить из БД:**
   - Нажмите кнопку "Все" в "Статистика за период"
   - Смотрите STEP логи в консоли
   - Проверьте заполнилась ли select

## 🚩 Возможные причины проблемы

1. **Completions не сохраняются:**
   - Нет данных в БД
   - API возвращает ошибку при сохранении
   - 👉 Проверка: `/api/debug/list-completions` должен показать результаты

2. **Дата не совпадает при поиске:**
   - JavaScript отправляет дату в неправильном формате
   - БД ожидает другой формат
   - 👉 Проверка: в консоли посмотреть что отправляется в STEP 1 loadDayFromDB

3. **Таблица completions не создана:**
   - Database migration не сработала
   - 👉 Проверка: при запуске app.py должны быть логи "[DEBUG] Creating table for completions"

4. **API не возвращает данные:**
   - Endpoint не сработал
   - 👉 Проверка: STEP 2 в loadDatesFromDB должна показать response

## 📝 Последующие шаги

После диагностики:
1. Поделитесь логами из браузер консоли
2. Выполните `/api/debug/list-completions` и покажите результат
3. Укажите что видно в STEP логах при сохранении по "Сохранить в БД"

Это поможет точно определить где происходит сбой в цепи обработки.
