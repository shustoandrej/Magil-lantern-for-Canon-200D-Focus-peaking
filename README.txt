Magic Lantern для Canon 200D (прошивка 1.0.1) - ИСПРАВЛЕННАЯ сборка (focus peaking работает)
Дата сборки: 2026-10-13
Версия: 2026-10-13.200D.101

RU
==
Что исправлено
---------------
В предыдущей сборке пункт "Focus Peak" в меню Overlay присутствовал, но на экране
ничего не рисовалось. Причина найдена в коде:

* на камерах с FEATURE_VRAM_RGBA (DIGIC 6/7, к которым относится 200D) Magic Lantern
  рисует свои экранные оверлеи в индексированный буфер, который возвращает функция
  bmp_vram();
* функция refresh_yuv_from_rgb() каждый кадр ЗАНОВО собирает RGBA-слой Canon из этого
  индексированного буфера;
* рисование зебр и фокусировочного пикинга шло в ДРУГОЙ буфер - bmp_vram_real()
  (напрямую в RGBA-слой Canon), поэтому всё нарисованное стиралось на следующем кадре;
* при этом стирание (clrscr_mirror) и другие оверлеи (кропмарки) уже работали через
  bmp_vram() - то есть сама логика была противоречива.

На не-RGBA камерах (500D/550D/600D/5D2/5D3 и т.п.) bmp_vram() и bmp_vram_real()
указывают на один и тот же буфер, поэтому там проблема не проявлялась и пикинг
работал. Именно поэтому баг не был виден на старых камерах.

Что сделано
-----------
1. В src/zebra.c введён макрос zebras_bvram():
   - на FEATURE_VRAM_RGBA -> bmp_vram()      (индексированный, композитируемый буфер);
   - на остальных        -> bmp_vram_real()  (поведение не изменено).
2. Этим макросом заменены три места рисования: draw_zebras_raw_lv(),
   draw_zebras(), draw_zebra_and_focus().
3. В путях рисования зебр и пикинга на RGBA-камерах выставляется флаг
   ml_refresh_display_needed = 1, чтобы композитор вывел нарисованное на экран.

Изменения на не-RGBA камерах отсутствуют полностью (макрос и флаг закрыты
#ifdef FEATURE_VRAM_RGBA - ветка для них в точности прежняя).

Побочно этим же исправлением начинают работать и зебры (Overlay -> Zebras /
"Overexposure"), т.к. они рисовались тем же сломанным способом.

Состав
------
autoexec.bin     - сама надстройка (ЭТО ЕДИНСТВЕННЫЙ ИЗМЕНЁННЫЙ ФАЙЛ)
ML/              - файлы Magic Lantern (модули, скрипты, шрифты)
ML-SETUP.FIR     - установщик первого запуска (кладётся в корень карты)
magiclantern.zip - та же сборка одним архивом (на случай установки "с нуля")

Как поставить (если ML уже установлен)
--------------------------------------
1. Выключить камеру, вынуть карту, вставить в компьютер.
2. Заменить файл autoexec.bin в корне карты на файл из этой папки.
   (Папку ML и ML-SETUP.FIR можно не трогать - они не менялись.
    Если хотите - перезапишите и папку ML из этой папки.)
3. Вставить карту, включить камеру, запустить ML (кнопка/пункт как обычно).
4. Меню ML -> вкладка Overlay -> "Focus Peak" -> выбрать цвет (например Red).
   Убедиться, что "Global Draw" в той же вкладке включён (по умолчанию ON).
5. В Live View на резких краях должны появиться цветные точки.

Важно / честно
--------------
* Исправление проверено анализом кода и инспекцией собранного бинарника:
  объектный файл zebra.o теперь ссылается на bmp_vram (а не bmp_vram_real),
  сборка проходит без ошибок (make: 0), хеш autoexec.bin изменился.
* На реальной камере 200D эта сборка НЕ запускалась (камеры под рукой нет).
  Проверка на живом железе - за вами. Если пикинг всё ещё не виден, сообщите -
  добавим диагностический вывод значений буфера прямо на экран.


EN
==
This build fixes focus peaking on the Canon 200D (a FEATURE_VRAM_RGBA camera).

Root cause: zebras / focus peaking were drawn into bmp_vram_real() (Canon's RGBA
layer), which refresh_yuv_from_rgb() fully rebuilds from the indexed buffer every
frame, so the drawing was erased immediately. The erase path (clrscr_mirror) and
cropmarks already used bmp_vram() (the indexed composited buffer). On non-RGBA
cameras both calls return the same buffer, which is why the bug only shows on
newer (DIGIC 6/7) bodies like the 200D.

Fix: a zebras_bvram() macro that picks bmp_vram() on FEATURE_VRAM_RGBA and keeps
bmp_vram_real() elsewhere; used at the three draw sites; plus an explicit
ml_refresh_display_needed = 1 on the RGBA draw paths. Non-RGBA behaviour is
byte-for-byte unchanged.

Only autoexec.bin changed; replace it in the card root.

Verified by code analysis and binary/object inspection (zebra.o references
bmp_vram, build succeeds, autoexec.bin hash changed). NOT run on real 200D
hardware.
