Возникла ошибка при сохранении файла. Вот содержимое документации, которое вы можете скопировать и вставить в документ Word:

---

### Документация для хранимых процедур, триггеров, функций и представлений

#### Хранимые процедуры

##### 1. Хранимая процедура: `add_new_post`

**Описание:**  
Эта процедура добавляет новый пост в таблицу `posts`.

**Полезность:**  
Она инкапсулирует логику добавления нового поста, обеспечивая правильное заполнение полей и упрощая процесс для пользователей и приложений, взаимодействующих с базой данных.

**Причина для включения:**  
Добавление постов - это общая операция в контексте социальных сетей, и наличие специальной процедуры для этой задачи помогает поддерживать целостность данных и уменьшает количество повторяющегося кода.

```sql
CREATE OR REPLACE PROCEDURE add_new_post(
    p_channel_id INT,
    p_source VARCHAR,
    p_date VARCHAR
)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO posts (channel_id, source, date)
    VALUES (p_channel_id, p_source, p_date);
END;
$$;
```

##### 2. Хранимая процедура: `update_channel_followers`

**Описание:**  
Эта процедура обновляет количество подписчиков для каждого канала на основе количества пользователей, связанных с этим каналом.

**Полезность:**  
Она обеспечивает актуальность и точность количества подписчиков в таблице `channels`.

**Причина для включения:**  
Точные данные о количестве подписчиков важны для аналитики и отчетности. Эта процедура помогает поддерживать точность данных.

```sql
CREATE OR REPLACE PROCEDURE update_channel_followers()
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE channels
    SET followers = (SELECT COUNT(*)
                     FROM channel_users
                     WHERE channel_users.channel_id = channels.id);
END;
$$;
```

##### 3. Хранимая процедура: `delete_user_and_related_data`

**Описание:**  
Эта процедура удаляет пользователя и все связанные с ним данные из нескольких таблиц (`comments`, `user_reaction`, `users_online`).

**Полезность:**  
Она обеспечивает, что при удалении пользователя не остаются осиротевшие записи в базе данных, поддерживая ссылочную целостность.

**Причина для включения:**  
Удаление пользователей - критическая операция, и эта процедура автоматизирует процесс, уменьшая риск ошибок и обеспечивая согласованность данных.

```sql
CREATE OR REPLACE PROCEDURE delete_user_and_related_data(p_user_id INT)
LANGUAGE plpgsql
AS $$
BEGIN
    DELETE FROM comments WHERE user_id = p_user_id;
    DELETE FROM user_reaction WHERE user_id = p_user_id;
    DELETE FROM users_online WHERE user_id = p_user_id;
    DELETE FROM users WHERE id = p_user_id;
END;
$$;
```

##### 4. Хранимая процедура: `add_new_channel`

**Описание:**  
Эта процедура добавляет новый канал в таблицу `channels` и включает обработку ошибок и управление транзакциями.

**Полезность:**  
Она инкапсулирует логику добавления нового канала, обеспечивая атомарность операции и правильную обработку ошибок.

**Причина для включения:**  
Добавление каналов - это фундаментальная операция в контексте социальных сетей. Эта процедура обеспечивает правильное добавление каналов и поддерживает целостность базы данных даже в случае ошибки.

```sql
CREATE OR REPLACE PROCEDURE add_new_channel(
    p_name VARCHAR,
    p_link VARCHAR,
    p_followers INT
)
LANGUAGE plpgsql
AS $$
BEGIN
    BEGIN
        -- Start a transaction
        BEGIN TRANSACTION;

        -- Insert new channel
        INSERT INTO channels (name, link, followers)
        VALUES (p_name, p_link, p_followers);

        -- Commit the transaction
        COMMIT TRANSACTION;
    EXCEPTION
        WHEN others THEN
            -- Rollback transaction in case of error
            ROLLBACK TRANSACTION;
            RAISE;
    END;
END;
$$;
```

##### 5. Хранимая процедура: `update_username`

**Описание:**  
Эта процедура обновляет имя пользователя для указанного пользователя.

**Полезность:**  
Она предоставляет простой и безопасный способ обновления имен пользователей, гарантируя, что изменяется правильная запись пользователя.

**Причина для включения:**  
Имя пользователя может быть обновлено по различным причинам (например, по желанию пользователя, исправления). Эта процедура упрощает процесс и обеспечивает согласованность.

```sql
CREATE OR REPLACE PROCEDURE update_username(
    p_user_id INT,
    p_new_username VARCHAR
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE users
    SET username = p_new_username
    WHERE id = p_user_id;
END;
$$;
```

#### Триггер

##### Триггер: `update_followers_on_new_user`

**Описание:**  
Этот триггер обновляет количество подписчиков канала всякий раз, когда в канал добавляется новый пользователь.

**Полезность:**  
Он обеспечивает актуальность количества подписчиков без необходимости ручных обновлений или периодических пакетных заданий.

**Причина для включения:**  
Поддержание актуальности количества подписчиков в реальном времени улучшает точность аналитики и пользовательский опыт.

```sql
CREATE OR REPLACE FUNCTION update_followers_on_new_user()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE channels
    SET followers = followers + 1
    WHERE id = NEW.channel_id;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trigger_update_followers
AFTER INSERT ON channel_users
FOR EACH ROW
EXECUTE FUNCTION update_followers_on_new_user();
```

#### Функции

##### Функция: `get_followers_count`

**Описание:**  
Эта функция возвращает количество подписчиков для указанного канала.

**Полезность:**  
Она предоставляет простой способ получения количества подписчиков для канала, что полезно для аналитики и отчетности.

**Причина для включения:**  
Часто требуется в отчетах и пользовательских интерфейсах для отображения количества подписчиков канала.

```sql
CREATE OR REPLACE FUNCTION get_followers_count(p_channel_id INT)
RETURNS INT
LANGUAGE plpgsql
AS $$
DECLARE
    followers_count INT;
BEGIN
    SELECT followers INTO followers_count
    FROM channels
    WHERE id = p_channel_id;
    RETURN followers_count;
END;
$$;
```

##### Функция: `get_posts_by_channel`

**Описание:**  
Эта функция возвращает все посты для указанного канала.

**Полезность:**  
Она упрощает получение постов для канала, что облегчает создание отчетов и отображение постов в пользовательских интерфейсах.

**Причина для включения:**  
Часто требуется для отображения постов в пользовательских интерфейсах и создания отчетов по содержимому.

```sql
CREATE OR REPLACE FUNCTION get_posts_by_channel(p_channel_id INT)
RETURNS TABLE(id INT, channel_id INT, source VARCHAR, date VARCHAR)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT id, channel_id, source, date
    FROM posts
    WHERE channel_id = p_channel_id;
END;
$$;
```

#### Представления

##### Представление: `user_activity_report`

**Описание:**  
Это представление предоставляет сводный отчет о активности пользователей, включая количество комментариев и реакций каждого пользователя.

**Полезность:**  
Оно помогает в создании отчетов и аналитики по вовлеченности и уровню активности пользователей.

**Причина для включения:**  
Полезно для администраторов и аналитиков для мониторинга вовлеченности пользователей и выявления активных пользователей.

```sql
CREATE OR REPLACE VIEW user_activity_report AS
SELECT u.id, u.username, COUNT(c.id) AS comment_count, COUNT(ur.id) AS reaction_count
FROM users u
LEFT JOIN comments c ON u.id = c.user_id
LEFT JOIN user_reaction ur ON u.id = ur.user_id
GROUP BY u.id, u.username;
```

##### Представление: `post_comment_report`

**Описание:**  
Это представление предоставляет сводный отчет о постах и количестве комментариев к ним.

**Полезность:**  
Оно помогает в создании отчетов и аналитики по вовлеченности и уровню активности по постам.

**Причина для включения:**  
Полезно для администраторов и аналитиков для мониторинга вовлеченности по постам и выявления популярного контента.

```sql
CREATE OR REPLACE VIEW post_comment_report AS
SELECT p.id AS post_id, p.channel_id, COUNT(c.id) AS comment_count
FROM posts p
LEFT JOIN comments c ON p.id = c.post_id
GROUP BY p.id, p.channel_id;
```

---

Скопируйте этот текст и вставьте его в документ Word. Сохраните документ с именем `database_documentation.docx`.
