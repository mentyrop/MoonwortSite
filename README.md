# Moonwort — лендинг

Статический одностраничный лендинг проекта Moonwort. Домен: **moonwort.ru**.

## Что где

| Файл | Назначение |
|------|------------|
| `index.html` | **Продакшн-лендинг** |
| `moonwort-mark.svg` | Эталонный векторный знак Moonwort из фирменного исходника |
| `moonwort-mark-shadow.png` | Мягкая тень знака, извлечённая из Illustrator/PDF |
| `README.md` | Краткое описание проекта и локальный запуск |
| `DEPLOY.md` | Инструкция по размещению на `moonwort.ru` |

Лендинг состоит из HTML и двух файлов фирменного знака; шрифты подключаются с
Google Fonts, фавикон встроен как data-URI. Сборка не нужна.

## Локальный просмотр

```bash
python3 -m http.server 8742
# открыть http://localhost:8742/index.html
```

## Деплой

См. [DEPLOY.md](./DEPLOY.md).
