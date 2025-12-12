## Part #1 Создание схемы и таблиц
```
DROP SCHEMA IF EXISTS info21 CASCADE;
CREATE SCHEMA info21;
SET search_path TO info21;

-- ENUM для статусов проверки
CREATE TYPE check_status AS ENUM ('Start', 'Success', 'Failure');

-- Таблица Peers
CREATE TABLE Peers (
    id SERIAL PRIMARY KEY,
    nickname VARCHAR(50) UNIQUE NOT NULL
);

-- Таблица Tasks
CREATE TABLE Tasks (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) UNIQUE NOT NULL,
    max_xp INTEGER NOT NULL
);

-- Таблица Checks
CREATE TABLE Checks (
    id SERIAL PRIMARY KEY,
    peer_id INTEGER REFERENCES Peers(id),
    task_id INTEGER REFERENCES Tasks(id),
    date TIMESTAMP DEFAULT NOW()
);

-- P2P проверки
CREATE TABLE P2P (
    id SERIAL PRIMARY KEY,
    check_id INTEGER REFERENCES Checks(id) ON DELETE CASCADE,
    checking_peer INTEGER REFERENCES Peers(id),
    status check_status,
    time TIMESTAMP DEFAULT NOW()
);

-- Verter проверки
CREATE TABLE Verter (
    id SERIAL PRIMARY KEY,
    check_id INTEGER REFERENCES Checks(id) ON DELETE CASCADE,
    status check_status,
    time TIMESTAMP DEFAULT NOW()
);

-- Трансферы P2P баллов
CREATE TABLE TransferredPoints (
    id SERIAL PRIMARY KEY,
    checking_peer INTEGER REFERENCES Peers(id),
    checked_peer INTEGER REFERENCES Peers(id),
    points INTEGER NOT NULL
);

-- Дружба
CREATE TABLE Friends (
    peer1 INTEGER REFERENCES Peers(id),
    peer2 INTEGER REFERENCES Peers(id),
    PRIMARY KEY (peer1, peer2)
);

-- Рекомендации
CREATE TABLE Recommendations (
    id SERIAL PRIMARY KEY,
    peer_id INTEGER REFERENCES Peers(id),
    recommended_peer INTEGER REFERENCES Peers(id)
);

-- XP
CREATE TABLE XP (
    id SERIAL PRIMARY KEY,
    check_id INTEGER REFERENCES Checks(id),
    xp_amount INTEGER NOT NULL
);

-- Отслеживание времени присутствия
CREATE TABLE TimeTracking (
    id SERIAL PRIMARY KEY,
    peer_id INTEGER REFERENCES Peers(id),
    visit_time TIMESTAMP DEFAULT NOW(),
    state VARCHAR(10) CHECK (state IN ('in', 'out'))
);
```
#### Таблица checks
<img width="471" height="535" alt="image" src="https://github.com/user-attachments/assets/7a1be88e-dbd2-4532-aabf-0727ab2f27a2" />

#### Таблица friends
<img width="316" height="540" alt="image" src="https://github.com/user-attachments/assets/a40ff67d-1eab-4753-be30-bc93bb5efea7" />

#### Таблица p2p
<img width="655" height="540" alt="image" src="https://github.com/user-attachments/assets/61006df7-2a94-4387-b853-7b1f8293aa74" />

#### Таблица peers
<img width="291" height="539" alt="image" src="https://github.com/user-attachments/assets/6f5685d4-4f4f-4e01-a0b2-386591defb6f" />

#### Таблица recommendations
<img width="368" height="539" alt="image" src="https://github.com/user-attachments/assets/1fa57d92-d8c2-4ec4-a7cc-8ce38b0d8fd4" />

#### Таблица tasks
<img width="371" height="541" alt="image" src="https://github.com/user-attachments/assets/320db1da-06e3-48d3-b2fd-ae5dcaa61049" />

#### Таблица timetracking
<img width="554" height="537" alt="image" src="https://github.com/user-attachments/assets/3d9f7fd7-fb82-4fd4-aa31-7a87a7acf1aa" />

#### Таблица transferredpoints
<img width="435" height="544" alt="image" src="https://github.com/user-attachments/assets/2c788331-8eba-4a2a-a4ed-54000b364887" />

#### Таблица verter
<img width="552" height="542" alt="image" src="https://github.com/user-attachments/assets/eb879ee7-639d-4fc6-af9f-ea51ee006f0e" />

#### Таблица xp
<img width="332" height="544" alt="image" src="https://github.com/user-attachments/assets/0f154ca4-7fb5-4fe3-b961-ff151b86c11c" />

## Part #2 Процедуры + триггеры
#### 1. Процедура: добавление P2P проверки:
```
CREATE OR REPLACE PROCEDURE add_p2p_check(
    p_checked_peer VARCHAR,
    p_task VARCHAR,
    p_checking_peer VARCHAR,
    p_status check_status
)
LANGUAGE plpgsql
AS $$
DECLARE
    checked_id INT;
    checker_id INT;
    task_id INT;
    check_row_id INT;
BEGIN
    SELECT id INTO checked_id FROM Peers WHERE nickname = p_checked_peer;
    SELECT id INTO checker_id FROM Peers WHERE nickname = p_checking_peer;
    SELECT id INTO task_id FROM Tasks WHERE title = p_task;

    INSERT INTO Checks (peer_id, task_id)
    VALUES (checked_id, task_id)
    RETURNING id INTO check_row_id;

    INSERT INTO P2P (check_id, checking_peer, status)
    VALUES (check_row_id, checker_id, p_status);
END;
$$;
```
<img width="491" height="714" alt="image" src="https://github.com/user-attachments/assets/51cbb3e3-0a28-45b2-9d2b-9155b3eda5b2" />

<img width="620" height="710" alt="image" src="https://github.com/user-attachments/assets/0f7eef99-4991-4847-83af-f889e270bb2b" />

#### 2. Процедура: добавление проверки Verter:
```
CREATE OR REPLACE PROCEDURE add_verter_check(
    p_checked_peer VARCHAR,
    p_task VARCHAR,
    p_status check_status
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_peer_id INT;
    v_task_id INT;
    v_check_id INT;
BEGIN
    -- найти ID пира
    SELECT id INTO v_peer_id 
    FROM Peers 
    WHERE nickname = p_checked_peer;

    -- найти ID задачи
    SELECT id INTO v_task_id
    FROM Tasks 
    WHERE title = p_task;

    -- найти последнюю проверку этого пира по этой задаче
    SELECT id INTO v_check_id
    FROM Checks
    WHERE Checks.peer_id = v_peer_id
      AND Checks.task_id = v_task_id
    ORDER BY date DESC
    LIMIT 1;

    -- вставить проверку Verter
    INSERT INTO Verter (check_id, status)
    VALUES (v_check_id, p_status);
END;
$$;
```
<img width="539" height="711" alt="image" src="https://github.com/user-attachments/assets/245557ab-7ba4-4d6d-a341-65a7e2f043ee" />

#### 3. Триггер: ограничение изменения P2P записей:
```
CREATE OR REPLACE FUNCTION lock_p2p_changes()
RETURNS trigger AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        RAISE EXCEPTION 'P2P records cannot be modified';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER p2p_immutable
BEFORE UPDATE ON P2P
FOR EACH ROW
EXECUTE FUNCTION lock_p2p_changes();
```
<img width="628" height="710" alt="image" src="https://github.com/user-attachments/assets/93ec0b52-7fdf-4a5c-b897-cc4bae8f37cd" />

Пробуем изменить запись

<img width="542" height="720" alt="image" src="https://github.com/user-attachments/assets/144ce20d-0404-4c25-8ac2-8ba9a5ce9379" />

Выдает ошибку, значит триггер работает правильно
#### 4. Триггер: автоматическая выдача XP после успешной проверки:
```
CREATE OR REPLACE FUNCTION give_xp_after_success()
RETURNS trigger AS $$
DECLARE
    task_max_xp INT;
    peer_task INT;
BEGIN
    SELECT task_id INTO peer_task FROM Checks WHERE id = NEW.check_id;
    SELECT max_xp INTO task_max_xp FROM Tasks WHERE id = peer_task;

    IF NEW.status = 'Success' THEN
        INSERT INTO XP (check_id, xp_amount)
        VALUES (NEW.check_id, task_max_xp);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER verter_success_xp
AFTER INSERT ON Verter
FOR EACH ROW
EXECUTE FUNCTION give_xp_after_success();
```
<img width="460" height="708" alt="image" src="https://github.com/user-attachments/assets/6402b37b-26d5-4038-b4bd-81af3662baf3" />
Добавляем проверку Verter
<img width="377" height="678" alt="image" src="https://github.com/user-attachments/assets/6a5dc5e8-237a-411c-bd45-af0a85c1ef81" />
Проверяем таблицу XP
<img width="400" height="707" alt="image" src="https://github.com/user-attachments/assets/b83aa1fa-97b9-4afd-a3aa-1ca22c811f67" />

## Part #3 Функции получения данных
#### 1. Перенесенные точки (TransferredPoints) в удобочитаемом формате
```
CREATE OR REPLACE FUNCTION get_transferred_points()
RETURNS TABLE (
    from_peer VARCHAR,
    to_peer VARCHAR,
    points INTEGER
)
AS $$
BEGIN
    RETURN QUERY
    SELECT
        p1.nickname,
        p2.nickname,
        tp.points
    FROM TransferredPoints tp
    JOIN Peers p1 ON tp.checking_peer = p1.id
    JOIN Peers p2 ON tp.checked_peer = p2.id;
END;
$$ LANGUAGE plpgsql;
```
<img width="392" height="710" alt="image" src="https://github.com/user-attachments/assets/089a7af2-40e8-4093-b613-932171be4e9b" />

#### 2. XP по пользователям и задачам
```
CREATE OR REPLACE FUNCTION get_user_xp()
RETURNS TABLE (
    peer VARCHAR,
    task VARCHAR,
    xp INTEGER
)
AS $$
BEGIN
    RETURN QUERY
    SELECT
        Peers.nickname,
        Tasks.title,
        XP.xp_amount
    FROM XP
    JOIN Checks ON XP.check_id = Checks.id
    JOIN Peers ON Checks.peer_id = Peers.id
    JOIN Tasks ON Checks.task_id = Tasks.id;
END;
$$ LANGUAGE plpgsql;
```
<img width="414" height="760" alt="image" src="https://github.com/user-attachments/assets/16f4f6db-6ebe-4885-afc5-878aed1755bb" />

#### 3. Пиры, которые не покидали кампус весь день
```
CREATE OR REPLACE FUNCTION peers_never_left_day(p_date DATE)
RETURNS TABLE (peer VARCHAR)
AS $$
BEGIN
    RETURN QUERY
    SELECT nickname
    FROM Peers
    WHERE id IN (
        SELECT peer_id
        FROM TimeTracking
        GROUP BY peer_id
        HAVING SUM(CASE WHEN state = 'out' THEN 1 ELSE 0 END) = 0
               AND DATE(MIN(visit_time)) = p_date
    );
END;
$$ LANGUAGE plpgsql;
```
<img width="581" height="161" alt="image" src="https://github.com/user-attachments/assets/6e359f05-fbed-41f0-a3e6-5cd0c14d9ef5" />

<img width="477" height="713" alt="image" src="https://github.com/user-attachments/assets/b62609ee-4a7d-4ee0-ade7-990af57b3bb7" />

#### 4. Баланс P2P-поинтов
```
CREATE OR REPLACE FUNCTION peer_points_balance()
RETURNS TABLE (
    peer VARCHAR,
    balance INTEGER
)
AS $$
BEGIN
    RETURN QUERY
    SELECT
        p.nickname,
        COALESCE(SUM(
            CASE
                WHEN tp.checking_peer = p.id THEN tp.points * -1
                WHEN tp.checked_peer  = p.id THEN tp.points
            END
        ), 0) AS balance
    FROM Peers p
    LEFT JOIN TransferredPoints tp
        ON p.id IN (tp.checking_peer, tp.checked_peer)
    GROUP BY p.nickname;
END;
$$ LANGUAGE plpgsql;
```
<img width="393" height="810" alt="image" src="https://github.com/user-attachments/assets/6fe11271-5f6c-4f7b-a8e7-7fd674a999cc" />
