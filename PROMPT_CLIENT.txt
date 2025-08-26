Есть веб-сервис для рисования на доске через REST API по аналогии с reddit r/place.
Сделай очень простого клиента для этого REST API на PYTHON

Который будет рисовать на канве с помощью REST API этого сервера:
логотип из букв "Google" разных цветов.

Придумай рандомный кринжовый веселый но простой ник, зафиксируй его и используй для отправки в API.
Зафиксируй случайный оффсет по x и y (но чтобы умещалось все изображение).
Должно запускаться по кнопке.


СХЕМА API СЕРВИСА:

openapi: 3.1.0
info:
  title: Mini r/place Drawing API
  version: "1.0"
  description: |
    REST API для рисования пикселей на общей канве.
    Сервер применяет **rate limit 5 RPS** на пару *(IP, nickname)*.
    Цвета принимаются в форматах **#RRGGBB** или **rgb(r,g,b)** (0–255).
servers:
  - url: https://place.skogorev.com
paths:
  /api/pixel:
    post:
      summary: Поставить/перекрасить один пиксель
      description: |
        Рисует один пиксель на координатах *(x, y)* с указанным цветом от имени пользователя *nickname*.

        **Ограничения и валидность:**
        - 5 запросов в секунду на *(IP, nickname)* → при превышении вернётся **429**.
        - `x` в диапазоне `[0, width)`, `y` в диапазоне `[0, height)` — размеры узнавайте через **GET /api/meta**.
        - `color` в формате **#RRGGBB** или **rgb(r,g,b)** (каждый компонент 0–255).
        - `nickname` — непустая строка (1–40 символов), пробелы по краям обрезаются.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/PixelRequest"
            examples:
              hex:
                summary: Синий пиксель (HEX)
                value: { x: 10, y: 5, color: "#4285F4", nickname: "alice" }
              rgb:
                summary: Красный пиксель (RGB)
                value: { x: 0, y: 0, color: "rgb(255,0,0)", nickname: "bob" }
      responses:
        "200":
          description: Успех
          content:
            application/json:
              schema: { $ref: "#/components/schemas/OkResponse" }
              examples:
                ok: { value: { ok: true } }
        "400":
          description: Ошибка валидности (цвет/координаты/никнейм)
          content:
            application/json:
              schema: { $ref: "#/components/schemas/ErrorResponse" }
              examples:
                badColor: { value: { detail: "Unsupported color format. Use #RRGGBB or rgb(r,g,b)" } }
                oob: { value: { detail: "Coordinates out of bounds" } }
        "429":
          description: Превышен лимит 5 RPS на (IP, nickname)
          content:
            application/json:
              schema: { $ref: "#/components/schemas/ErrorResponse" }
              examples:
                rl: { value: { detail: "Rate limit 5 RPS exceeded" } }


  /api/meta:
    get:
      summary: Получить параметры канвы и лимиты
      description: Возвращает текущие размеры доски, лимит RPS и настройку окна активности.
      responses:
        "200":
          description: Информация о канве и лимитах
          content:
            application/json:
              schema: { $ref: "#/components/schemas/MetaResponse" }
              examples:
                meta:
                  value:
                    width: 256
                    height: 128
                    rps_limit: 5
                    active_window_sec: 20
                    clear_key_required: false


components:
  schemas:
    PixelRequest:
      type: object
      required: [x, y, color, nickname]
      properties:
        x:
          type: integer
          minimum: 0
          description: Координата X (левая граница = 0; правая граница = width-1)
        y:
          type: integer
          minimum: 0
          description: Координата Y (верхняя граница = 0; нижняя граница = height-1)
        color:
          type: string
          description: Цвет пикселя — формат **#RRGGBB** или **rgb(r,g,b)** (0–255)
          examples: ["#000000", "#FFFFFF", "rgb(66,133,244)"]
        nickname:
          type: string
          minLength: 1
          maxLength: 40
          description: Имя пользователя, от которого ставится пиксель
    OkResponse:
      type: object
      properties:
        ok:
          type: boolean
          example: true
    ErrorResponse:
      type: object
      properties:
        detail:
          type: string
          description: Сообщение об ошибке
    MetaResponse:
      type: object
      properties:
        width: { type: integer, description: Ширина канвы в пикселях }
        height: { type: integer, description: Высота канвы в пикселях }
        rps_limit: { type: integer, description: Лимит запросов в секунду на (IP, nickname) }
        active_window_sec: { type: integer, description: Окно активности для отображения никнеймов в онлайне }
        clear_key_required: { type: boolean, description: Требуется ли CLEAR_KEY для /api/clear }
