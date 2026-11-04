# OS Class — Firebase / GitHub Pages setup

## Исходный файл и изменения
Сохранены исходный дизайн, стили, теория, двуязычные тексты, пять групповых карточек, задания, тесты и таймер. В исходной версии коды групп проверялись в JavaScript, студенты использовали анонимный вход, ответы всех участников группы записывались в общий документ `answers/gN`, индивидуальных ответов не было, а доступ преподавателя не был защищён Firestore Rules. Это позволяло обходить интерфейсные ограничения и создавало риск перезаписи ответов.

Новая архитектура использует Cloud Function для проверки кода и выдачи custom token, Custom Claims для роли, отдельные коллекции `students`, `submissions`, `groupAnswers`, real-time `onSnapshot()` и строгие правила Firestore.

**Firebase пока не подключён к вашему проекту.** Нужно заменить конфигурацию и развернуть backend.

## Файлы
- `index.html` — сайт и Firebase client.
- `firestore.rules` — правила доступа.
- `firebase-functions/index.js` — серверная проверка группового кода.
- `firebase-functions/package.json` — зависимости Functions.
- `firebase.json` — конфигурация Firebase CLI.

## 1. Firebase config
Создайте проект Firebase, добавьте Web App в Project settings и скопируйте Web App config. В `index.html` найдите `window.FIREBASE_CONFIG` и замените все `PASTE_...` значениями из консоли. Включите Authentication → Email/Password и создайте Firestore Database. В Authentication → Settings → Authorized domains добавьте домен GitHub Pages.

Firebase web config не является административным секретом. Не помещайте в HTML service-account JSON, приватные ключи или коды групп.

## 2. Cloud Function и коды групп
Установите Firebase CLI, войдите и выберите проект:
```bash
npm install -g firebase-tools
firebase login
firebase use --add
```
Задайте секрет:
```bash
firebase functions:secrets:set GROUP_CODES_JSON
```
Вставьте значение:
```json
{"CS1-4821":1,"CS2-7350":2,"MA3-1946":3,"MA4-6082":4,"MA5-3379":5}
```
Задайте отдельный длинный случайный секрет:
```bash
firebase functions:secrets:set UID_HMAC_SECRET
```
Установите зависимости и разверните функцию:
```bash
cd firebase-functions
npm install
cd ..
firebase deploy --only functions
```
Функция требует App Check. Настройте App Check для Web App с reCAPTCHA v3, замените `PASTE_RECAPTCHA_V3_SITE_KEY` в `index.html` на публичный site key и добавьте домен сайта в разрешённые домены App Check. Инициализация App Check уже предусмотрена в клиентском модуле. Не отключайте `enforceAppCheck` в production.

### Ограничение идентификации студента
Имя + общий код группы не могут однозначно подтвердить личность или различить двух студентов с одинаковым именем. В этой реализации случайный browser key хранится в localStorage, а UID вычисляется сервером через HMAC. Повторный вход в том же браузере использует прежний UID; очистка данных браузера или другое устройство создаёт другой UID. Для строгой идентификации нужен персональный код/учётная запись каждого студента с серверной проверкой.

## 3. Преподаватель
Создайте пользователя преподавателя в Firebase Authentication. Затем из доверенной среды Admin SDK назначьте Custom Claims:
```js
const {initializeApp,applicationDefault}=require("firebase-admin/app");
const {getAuth}=require("firebase-admin/auth");
initializeApp({credential:applicationDefault(),projectId:"YOUR_PROJECT_ID"});
(async()=>{const u=await getAuth().getUserByEmail("teacher@example.com");await getAuth().setCustomUserClaims(u.uid,{role:"teacher",teacher:true});})();
```
Запускайте только в защищённой администраторской среде с Application Default Credentials и установленным `firebase-admin`. Не загружайте credentials в GitHub. Преподавателю нужно выйти и войти снова для обновления токена.

## 4. Firestore
Структура:
- `students/{uid}` — профиль, группа, активность, прогресс, статистика.
- `submissions/{uid}_{taskId}` — индивидуальный ответ.
- `groupAnswers/g{groupId}_{taskId}` — общий ответ конкретной группы.
- `assignments/{id}` — задания преподавателя.
- `groups/{id}` — настройки групп без секретных кодов.

Разверните правила:
```bash
firebase deploy --only firestore:rules
```
Студент читает собственный профиль/ответы и общие ответы только своей группы. Преподаватель с claims `role=teacher` и `teacher=true` имеет доступ ко всем группам.

## 5. GitHub Pages
Загрузите файлы проекта в репозиторий и включите Pages в Settings → Pages. Frontend использует HTTPS Firebase CDN modules и не требует Node.js на GitHub Pages. Cloud Functions работают на стороне Firebase. Добавьте домен опубликованного сайта в Authorized domains и App Check.

## 6. Поведение
- Heartbeat: примерно раз в 60 секунд.
- Онлайн: активность не старше 150 секунд.
- `onSnapshot()` обновляет dashboard без перезагрузки.
- Индивидуальные и групповые ответы разделены.
- Повторная отправка студентом того же задания обновляет тот же документ.
- Архивация меняет `status` профиля, не удаляя Auth-пользователя и ответы.
- В фоновой вкладке браузер может задерживать heartbeat.

## 7. Проверка перед занятием
1. Вставьте Firebase config.
2. Разверните Functions и Firestore Rules.
3. Настройте App Check.
4. Назначьте преподавателю custom claims.
5. Проверьте пять кодов и один неверный.
6. Проверьте, что студент одной группы не читает данные другой.
7. Проверьте dashboard, поиск, фильтр, сортировку и архивацию.
8. Обновите страницу и проверьте восстановление сессии.
9. Проверьте работу без интернета и сообщения об ошибках.
10. Проверьте сайт по реальному URL GitHub Pages.

Я не подключал этот код к вашему Firebase-проекту и не выполнял live-интеграционные тесты. Перед использованием с классом выполните проверочный список выше.
