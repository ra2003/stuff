# amff2gif

Обёртка над ffmpeg, упрощающая создание гифок.

Использование с параметрами по умолчанию:

    ./amff2gif.py video.mp4 output.gif

Создаст 255-цветную гифку из видео с исходными размером и частотой
и с sierra2_4a dithering mode.

Параметром `-m` можно задать число цветов, параметрами `-W` и `-H` — размеры
(если указать только W или H, другая сторона автоматически вычислится
с сохранением соотношения сторон), `-r` изменяет частоту кадров. Больше
параметров можно увидеть в `./amff2gif.py --help`
