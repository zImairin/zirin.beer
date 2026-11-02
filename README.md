<div align="center">

# zirin.beer

Плохой UX здесь не ошибка, попробуйте завайбиться.

[Открыть сайт](https://zirin.beer)

<br>

![Svelte 5](https://img.shields.io/badge/Svelte-5-FF3E00?style=flat-square&logo=svelte&logoColor=white)
![SvelteKit](https://img.shields.io/badge/SvelteKit-Static-FF3E00?style=flat-square&logo=svelte&logoColor=white)
![Three.js](https://img.shields.io/badge/Three.js-3D-111111?style=flat-square&logo=threedotjs&logoColor=white)
![GitHub Pages](https://img.shields.io/badge/Deploy-GitHub%20Pages-222222?style=flat-square&logo=githubpages&logoColor=white)

</div>

---

## Маршрут

| Этап  | Страница                                   | Что происходит                                                                                                     |
| ----- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| 01    | [`/`](https://zirin.beer/)                 | Хаотичные курсоры окружают кнопку. Нажатие запускает следующий этап.                                               |
| 02    | [`/maze`](https://zirin.beer/maze)         | Пользователь сам проходит лабиринт. При столкновении со стеной след попытки сбрасывается.                          |
| 03    | [`/ads`](https://zirin.beer/ads)           | Поверх старого сайта появляются окна с предложениями скачать Яндекс Браузер и MAX. Их крестики убегают или падают. |
| Финиш | [`???`](https://zirin.beer/shawarma)       | ?                                                                                                                  |

## Запуск локально

Требуется Node.js 22 или новее.

```bash
git clone https://github.com/zImairin/zirin.beer.git
cd zirin.beer
npm ci
npm run dev
```

Vite выведет локальный адрес в терминал.

## Команды

| Команда           | Назначение                             |
| ----------------- | -------------------------------------- |
| `npm run dev`     | Запустить сайт для разработки          |
| `npm run check`   | Проверить типы и компоненты Svelte     |
| `npm run build`   | Собрать статический сайт в `build/`    |
| `npm run preview` | Просмотреть production-сборку локально |

## Публикация

GitHub Actions собирает сайт и публикует содержимое `build/` при каждом push в `main`. Workflow находится в `.github/workflows/deploy.yml`, а домен `zirin.beer` задаётся файлом `static/CNAME`.

## Технологии

- Svelte 5 и SvelteKit
- TypeScript и Vite
- Three.js через Threlte для 3D-сцены
- `adapter-static` и GitHub Pages для публикации
