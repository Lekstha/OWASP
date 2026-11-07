# Broken Access Control — Unprotected admin functionality with unpredictable URL

| | |
|---|---|
| **Лаба** | [Unprotected admin functionality with unpredictable URL](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality-with-unpredictable-url) |
| **Уровень** | Apprentice |
| **OWASP** | A01:2021 — Broken Access Control |
| **CWE** | CWE-862: Missing Authorization |
| **Дата** | 2026-11-07 |
| **Инструменты** | браузер (DevTools), curl |

---

## 1. Цель

Получить доступ к панели администратора, путь к которой заранее неизвестен,
и удалить пользователя `carlos`.

---

## 2. Уязвимость

Маршрут администратора имеет непредсказуемое имя (`/admin-<случайный суффикс>`),
но по-прежнему **не проверяет права** при обращении.

Путь раскрывается в клиентском JavaScript главной страницы:

```javascript
var isAdmin = false;
if (isAdmin) {
   var topLinksTag = document.getElementsByClassName("top-links")[0];
   var adminPanelTag = document.createElement('a');
   adminPanelTag.setAttribute('href', '/admin-qjcdhs');
   adminPanelTag.innerText = 'Admin panel';
   topLinksTag.append(adminPanelTag);
}
```

Ссылка создаётся только при `isAdmin === true`, поэтому в интерфейсе её нет.
Но сам путь уже отправлен клиенту в тексте страницы.

---

## 3. Обнаружение

1. Открыл главную страницу → в UI ссылки на админку нет.
2. DevTools → View Source (или `Ctrl+U`) → поиск по слову `admin`.
3. Нашёл в JS переменную с путём `/admin-qjcdhs`.
4. Перешёл по этому пути без логина → `200`, панель отрисовалась.

---

## 4. Эксплуатация

```bash
curl -i https://<lab>/admin-qjcdhs
# HTTP/1.1 200 OK → панель отдана без аутентификации
```

В панели удалил пользователя `carlos`.

![Админ-панель без авторизации](img/002-admin-panel.png)

---

## 5. Почему работает

Здесь две ошибки, и вторая — главная:

1. **Утечка пути:** админ-URL отправлен клиенту в JS, хотя в UI не отображается.
   Всё, что попало в браузер, доступно пользователю.
2. **Отсутствие авторизации:** сервер отдаёт админ-панель любому, кто знает путь.

Непредсказуемый URL — это не контроль доступа, а надежда на то, что путь
не найдут. Скрытие кнопки в UI контроль тоже не заменяет.

---

## 6. Влияние

**В этой лабе:** удаление пользователей, полный контроль над админ-функциями.

**В общем случае:** чтение и изменение данных других пользователей, повышение
привилегий, создание администраторов, утечка PII.

---

## 7. Исправление (по приоритету)

1. **Серверная проверка роли** на каждом админ-маршруте — deny by default.
2. **Единый механизм авторизации** (middleware/декоратор).
3. **Не передавать админ-URL в клиентский код** — рендерить ссылку только
   на сервере и только для админа (defense in depth).
4. **Не полагаться на непредсказуемость URL.**
5. **Логировать** обращения к админ-маршрутам без прав.

---

## 8. Как это выглядит в коде

### Плохо

```python
# путь известен клиенту, проверки прав нет
@app.get("/admin-qjcdhs")
def admin_panel():
    return render_template("admin.html")
```

```javascript
// путь утекает в браузер
adminPanelTag.setAttribute('href', '/admin-qjcdhs');
```

### Хорошо

```python
@app.get("/admin-panel", dependencies=[Depends(require_admin)])
def admin_panel():
    return render_template("admin.html")
```

```python
# ссылка рендерится на сервере и только для админа
{% if current_user.role == "admin" %}
  <a href="/admin-panel">Admin panel</a>
{% endif %}
```

---

## 9. Вывод

Скрытый URL не защищает: путь виден в исходниках страницы и находится поиском
по слову `admin`. Разница с предыдущей лабой только в способе обнаружения —
механизм уязвимости тот же: сервер не проверяет права.

---

## 10. Что осталось непонятным

- [ ] Почему возвращать 403, а не 404? Что лучше скрывает существование ресурса
- [ ] Как автоматически искать админ-маршруты в JS-бандлах
