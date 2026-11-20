Лабораторна робота №6
Тема: Lab 06 - Іменовані набори: структури, переліки й купа

Студент: Свириденко Павло Володимирович
Група: Б-F2-26-2-ПІ

1. enum class Op
Константи OP_* з cpu.hpp замінено на enum class Op : std::uint8_t, значення ті самі, що в ISA.uk.md. step тепер перемикається по Op, а не по сирому байту. Невідомий байт, як і раніше, дає помилку "unknown opcode" і зупиняє процесор.

enum class Op : std::uint8_t {
    Halt = 0x00, Nop = 0x01, Out = 0x02, OutN = 0x03, OutS = 0x04,
    Add = 0x10, Sub = 0x11, And = 0x12, Or = 0x13, Xor = 0x14, Not = 0x15,
    Shl = 0x16, Shr = 0x17, Inc = 0x18, Dec = 0x19, Cmp = 0x1A,
    LoadIA = 0x20, LoadIB = 0x21, LoadA = 0x22, LoadB = 0x23,
    StoreA = 0x24, StoreB = 0x25, MovAB = 0x26, MovBA = 0x27,
    LoadH = 0x28, LoadAH = 0x29, StoreHA = 0x2A, IncH = 0x2B, DecH = 0x2C,
    HLow = 0x2D,
    Jmp = 0x30, Jz = 0x31, Jnz = 0x32,
    Cls = 0x40, Plot = 0x41, Show = 0x43
};

Поведінка не змінилась. Перевірка маленькою програмою (LOADI A 0x41, OUT, HALT):

ember> load 0 0x20 0x41 0x02 0x00
ember> reset
ember> run
A

2. Розкладка CPU
sizeof(CPU) = 24, sizeof(Flags) = 3, sizeof(Sprite) = 5. Виміряно програмою size_check.cpp (sizeof і offsetof).

поле     зсув   розмір
mem      0      8
pc       8      2
h        10     2
a        12     1
b        13     1
f        14     3
halted   17     1
(хвіст)  18-23  6

Поля займають байти 0-17, це 18 байтів, а sizeof дає 24. Шість байтів вирівнювальні, вони стоять у кінці. Причина: mem це вказівник на 8 байтів, тому вся структура вирівнюється по 8, і розмір округлюється до 24. Усередині структури дірок немає, кожне поле стоїть на своїй межі.

У Flags три bool по одному байту, 3 байти, вирівнювання по 1. У Sprite п'ять однобайтових полів, 5 байтів, padding немає.

3. Спрайти
struct Sprite { uint8_t x, y; int8_t vx, vy; bool alive; };
Sprite sprites[8] лежить у main.cpp. Команда sprite <i> <x> <y> <vx> <vy> заповнює один запис і ставить alive = true. Команда tick рухає всі живі спрайти, на межі екрана 64x32 змінює знак vx або vy, потім робить clear, plot для кожного спрайта і show.

ember> sprite 0 2 2 -1 -1
ember> sprite 1 61 5 1 0
ember> tick

Кадр 1: спрайт 0 в (1,1), спрайт 1 в (62,5).
Кадр 2: спрайт 0 в куті (0,0), спрайт 1 на правому краї, x = 63.
Кадр 3: обидва розвернулись, спрайт 0 знову в (1,1), спрайт 1 на x = 62.
Кадр 4: спрайт 0 в (2,2), спрайт 1 на x = 61.

4. Карта пам'яті
Усього 4096 байтів, адреси 0x000 - 0xFFF.

регіон          діапазон          розмір   що там
Код             0x000 - 0x7FF     2048     програма, PC стартує з 0x000
Дані            0x800 - 0x9FF     512      рядки, масиви, лічильники
Відеопам'ять    0xA00 - 0xAFF     256      екран 64x32, біт на піксель
Запас           0xB00 - 0xBFF     256      вільний
Купа            0xC00 - 0xEFF     768      зарезервована, поки ніхто не пише
Стек            0xF00 - 0xFFF     256      SP стартує з 0xFFF і йде вниз

Купа росте вгору від 0xC00, стек росте вниз від 0xFFF, тобто вони йдуть назустріч одне одному. В ember між ними стоїть перегородка на 0xF00, тому кожен закінчується окремо.

5. Санітайзер
Збірка: c++ -std=c++17 -g -fsanitize=address scratch.cpp -o scratch
Програма scratch.cpp робить int* p = new int{42}, а режим у аргументі вибирає, що з ним зробити. Платформа: Linux (Codespaces), тому витік показано справжнім звітом LeakSanitizer.

Витік (ASAN_OPTIONS=detect_leaks=1 ./scratch 0):
==8965==ERROR: LeakSanitizer: detected memory leaks
Direct leak of 4 byte(s) in 1 object(s) allocated from:
    #0 operator new(unsigned long)
    #1 main /workspaces/lab-05-standar/scratch.cpp:5
SUMMARY: AddressSanitizer: 4 byte(s) leaked in 1 allocation(s).

Об'єкт у купі живий, а вказівника на нього вже немає, тому звільнити його нікому.

Читання після звільнення (./scratch 1):
==35840==ERROR: AddressSanitizer: heap-use-after-free on address 0x502000000010 at pc 0x5ab687a1c3e2 bp 0x7ffccf9ef880 sp 0x7ffccf9ef870
READ of size 4 at 0x502000000010 thread T0
    #0 main /workspaces/lab-05-standar/scratch.cpp:8
SUMMARY: AddressSanitizer: heap-use-after-free /workspaces/lab-05-standar/scratch.cpp:8 in main

Після delete p вказівник лишився, а об'єкта вже немає. На рядку 8 програма читає з цього вказівника, ASan бачить, що ця пам'ять позначена як звільнена.

Подвійне звільнення (./scratch 2):
==9161==ERROR: AddressSanitizer: attempting double-free on 0x502000000010 in thread T0:
    #1 main /workspaces/lab-05-standar/scratch.cpp:9
freed by thread T0 here:
    #1 main /workspaces/lab-05-standar/scratch.cpp:7
previously allocated by thread T0 here:
    #1 main /workspaces/lab-05-standar/scratch.cpp:5
SUMMARY: AddressSanitizer: double-free

Перший delete на рядку 7, другий на рядку 9, обидва на одну адресу. Звіт показує три місця: де виділили, де звільнили, де звільнили вдруге.

6. Виправлення
Витік: додати delete p.
Читання після звільнення: після delete p писати p = nullptr і не читати через нього.
Подвійне звільнення: один delete на один new, після нього p = nullptr (delete від nullptr нічого не робить).