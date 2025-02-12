# Лабораторная работа №2

## Дано

1. [Текст на русском языке](assets/text.txt) (загружен и сохранен в переменную `text` в `start.py`).
2. Секретный зашифрованный текст
3. [Обученный на большом корпусе текстов словарь токенов](assets/vocab.json)
4. [Нейросетевая языковая модель](assets/nmt_demo) ([пример обращения](assets/nmt_demo/main.py))

По ходу выполнения лабораторной работы необходимо научиться предобрабатывать текст при
помощи алгоритма [Byte-Pair Encoding](https://en.wikipedia.org/wiki/Byte_pair_encoding)
таким образом, чтобы суметь получить предсказания от нейросетевой лингвистической модели
и оценить ее работу.

**Важно:** В рамках данной лабораторной работы **нельзя использовать сторонние модули, а также
стандартные модули collections и itertools.**

## Терминология

В данной лабораторной работе мы будем оперировать следующими понятиями:

* **слово** - последовательность символов между пробельными символами;
* **предобработанное слово** - слово, разбитое на токены;
* **частота слова** - количество вхождений слова в текст;
* **токен** - часть, выделяемая в слове;
* **идентификатор токена** - целочисленное значение, однозначно указывающее на определенный токен;
* **vocabulary (словарь токенов)** - словарь, устанавливающий однозначное соответствие между
токенами и идентификаторами токенов.

## Описание алгоритма `BPE` (Byte-Pair Encoding)

Целью автоматической обработки естественного языка (Natural Language Processing, NLP) является
формализация языка таким образом, чтобы сохранить его смысловое наполнение. Ключевую роль
в такой обработке играет токенизация и кодирование текста. Благодаря этому становится возможным
использование формальных языковых моделей, которые не способны работать с текстовыми данными
в первоначальном виде, но широко используются для выявления скрытых закономерностей в числовых
данных.

В качестве самого простого способа токенизировать текст можно рассмотреть разбиение текста на слова.
Однако у такого подхода есть существенные минусы: во-первых, размер словаря (vocabulary), то есть
множества всех уникальных токенов, в таком случае получается достаточно большим, во-вторых, даже
при таком очень большом словаре нередко приходится сталкиваться с проблемой
[Out-of-Vocabulary](https://blog.marketmuse.com/glossary/out-of-vocabulary-oov-definition/) слов.
Это не позволяет сделать обработку универсальной и масштабируемой на большое количество дискурсов.

По этой причине сегодня существует множество техник кодирования текста по частям слов. Одной из
них является Byte-Pair Encoding, с которым вам и предстоит поработать в рамках данной
лабораторной работы.

Данный алгоритм был предложен в 1994 Филиппом Гейджем в статье
["Новый метод сжатия данных"](http://www.pennelynn.com/Documents/CUJ/HTML/94HTML/19940045.HTM).
Идея алгоритма сводится к замене самых частотных пар символов другим символом, при этом объем
используемой памяти снижается в два раза.

В контексте токенизации текста BPE представляет собой итеративное
слияние наиболее частых пар последовательных токенов в тексте (или корпусе текстов)
до тех пор, пока не будет достигнут определенный размер словаря.

Для лучшего понимания рассмотрим применение этого алгоритма на примере.

Допустим, нам дан следующий текст:

```py
"It's far, farther, farthest and old, older, oldest"
```

Разобьем текст на слова по пробельным символам и посимвольно токенизируем каждое слово:

```py
[
    ('I', 't', "'", 's', '</s>'),
    ('f', 'a', 'r', ',', '</s>'),
    ('f', 'a', 'r', 't', 'h', 'e', 'r', ',', '</s>'),
    ('f', 'a', 'r', 't', 'h', 'e', 's', 't', '</s>'),
    ('a', 'n', 'd', '</s>'),
    ('o', 'l', 'd', ',', '</s>'),
    ('o', 'l', 'd', 'e', 'r', ',', '</s>'),
    ('o', 'l', 'd', 'e', 's', 't', '</s>')
]
```

Обратите внимание, что каждое слово заканчивается специальным токеном `'</s>'`,
обозначающим конец слова.

Составим таблицу частотности пар токенов. Нас интересуют те токены, которые идут друг за другом:

```py
{
    ('I', 't'): 1, ('t', "'"): 1, ("'", 's'): 1, ('s', '</s>'): 1,
    ('f', 'a'): 3, ('a', 'r'): 3, ('r', ','): 3, (',', '</s>'): 4,
    ('r', 't'): 2, ('t', 'h'): 2, ('h', 'e'): 2, ('e', 'r'): 2,
    ('e', 's'): 2, ('s', 't'): 2, ('t', '</s>'): 2,
    ('a', 'n'): 1, ('n', 'd'): 1, ('d', '</s>'): 1,
    ('o', 'l'): 3, ('l', 'd'): 3, ('d', ','): 1,
    ('d', 'e'): 2
}
```

Обратите внимание, что мы не рассматриваем пары токенов, встречающихся на границе слов.
Поэтому у нас нет пар, начинающихся с `</s>`.

Теперь необходимо выбрать самую часто встречающуюся пару. В нашем примере ею является пара
токенов `(',', '</s>')`. Давайте объединим ее в один токен:

```py
[
    ('I', 't', "'", 's', '</s>'),
    ('f', 'a', 'r', ',</s>'),
    ('f', 'a', 'r', 't', 'h', 'e', 'r', ',</s>'),
    ('f', 'a', 'r', 't', 'h', 'e', 's', 't', '</s>'),
    ('a', 'n', 'd', '</s>'),
    ('o', 'l', 'd', ',</s>'),
    ('o', 'l', 'd', 'e', 'r', ',</s>'),
    ('o', 'l', 'd', 'e', 's', 't', '</s>')
]
```

Далее мы пересчитываем встречаемость пар токенов, соединяем самую частую пару и повторяем процесс
до тех пор, пока не достигнем нужного размера словаря.

Итогом работы алгоритма должно стать следующее разбиение:

```py
[
    ('I', 't', "'", 's', '</s>'),
    ('far', ',</s>'),
    ('far', 'th', 'er', ',</s>'),
    ('far', 'th', 'est', '</s>'),
    ('a', 'n', 'd', '</s>'),
    ('old', ',</s>'),
    ('old', 'er', ',</s>'),
    ('old', 'est', '</s>')
]
```

Предполагается, что таким образом удается выделить из текста значимые последовательности: это
могут быть корни слов либо другие широко употребляемые морфемы. Так, в нашем примере нам удалось
выделить два корня (`far` и `old`), а также суффиксы сравнительной и превосходной степени `er` и `est`.

Иногда в алгоритме вместо специального токена конца слова используется токен, обозначающий,
напротив, начало слова. В настоящей лабораторной нам доведется столкнуться и с тем, и с другим.

Давайте пошагово рассмотрим реализацию такого алгоритма.
1. создание частотного словаря корпуса, отражающего количество вхождений каждого из слов;
2. токенизация слов (на первой итерации - разделение слов на символы);
3. составление частотного словаря пар токенов, которые следуют друг за другом в рамках одного слова;
4. формирование нового токена из самой часто встречающейся пары;
5. повторение шагов 2-4 до тех пор, пока не будет достигнуто желаемое количество токенов.

## Что надо сделать

### Шаг 0. Подготовка (проделать вместе с преподавателем на практике)

1. Изменить файлы `main.py` и `start.py`
2. Закоммитить изменения и создать новый pull request

**Важно**: Код, выполняющий все требуемые действия
должен быть написан в функции `main` в модуле `start.py`.
Для этого реализуйте функции в модуле `main.py` и импортируйте их в `start.py`.

```py
if __name__ == '__main__':
    main()
```

Обратите внимание, что в файле [`target_score.txt`](target_score.txt) необходимо выставить
желаемую оценку: 4, 6, 8 или 10. Чем выше желаемая оценка, тем большее количество тестов
запускается при проверке вашего Pull Request.

### Шаг 1. Токенизация одного слова

Начнем с предобработки слов.

Функция для токенизации одного слова принимает на вход строку и специальные токены,
обозначающие начало и конец слова.
Функция возвращает кортеж символов, начинающийся токеном начала слова и
оканчивающийся токеном конца слова. В случае, если какой-либо из специальных токенов представлен
значением `None`, то добавлять его в кортеж не требуется.

Функция не должна удалять из строки специальные символы или приводить текст к нижнему регистру.

Например, строка `"It's"` должна быть обработана следующим образом: `('I', 't', "'", 's', '</s>')`,
где `'</s>'` - токен, обозначающий конец слова, в качестве токена, обозначающего начало слова,
было передано `None`.

В случае, если на вход подаются некорректные значения (то есть не являющиеся строками или `None`),
функция возвращает `None`. Обратите внимание, что текст не может быть представлен как `None`.

Интерфейс:

```py
def prepare_word(
        raw_word: str, start_of_word: str | None, end_of_word: str | None
) -> tuple[str, ...] | None:
    pass
```

### Шаг 2. Формирование частотного словаря

### Выполнение Шага 2 соответствует 4 баллам

Далее необходимо собрать частотный словарь, ключами которого выступают предобработанные слова,
а значениями - количество вхождений слов в текст.
В рамках данной лабораторной работы будем придерживаться мнения, что граница слова - это
любой пробельный символ.

Например, из текста `"It's far, farther, farthest and old, older, oldest"` должен получиться
словарь следующего вида:

```py
{
    ('I', 't', "'", 's', '</s>'): 1,
    ('f', 'a', 'r', ',', '</s>'): 1,
    ('f', 'a', 'r', 't', 'h', 'e', 'r', ',', '</s>'): 1,
    ('f', 'a', 'r', 't', 'h', 'e', 's', 't', '</s>'): 1,
    ('a', 'n', 'd', '</s>'): 1,
    ('o', 'l', 'd', ',', '</s>'): 1,
    ('o', 'l', 'd', 'e', 'r', ',', '</s>'): 1,
    ('o', 'l', 'd', 'e', 's', 't', '</s>'): 1
}
```

Функция принимает на вход строку, по которой необходимо сформировать частотный словарь, а также
токены конца и начала слова. При этом токен начала слова может быть представлен `None`, а токен
конца - нет.

Функция обязательно должна вызывать функцию `prepare_word`. Если функция `prepare_word`
возвращает `None`, функция `collect_frequencies` так же должна вернуть `None`.

В случае, если на вход подаются некорректные значения (то есть значения с неверным типом),
функция должна возвращать `None`.

```py
def collect_frequencies(
        text: str, start_of_word: str | None, end_of_word: str
) -> dict[tuple[str, ...], int] | None:
    pass
```

Продемонстрируйте составление частотного словаря в функции `main()` модуля `start.py`, используя
текст на русском языке (переменная `text`). В качестве токена окончания слова используйте
строку `</s>`. Токен начала слова не используйте, то есть передайте в его качестве `None`.

### Шаг 3. Подсчет количества вхождений каждой из пар токенов

Для формирования новых токенов необходимо выделить самые часто встречающиеся сочетания уже
существующих токенов. Для этого нужно сформировать частотный словарь, в котором в качестве ключей
используются пары существующих токенов (то есть пока что только пары символов), а значениями -
количество случаев, когда эти символы следуют друг за другом в пределах одного слова.

Так, для примера из предыдущего шага частотный словарь сочетания токенов имеет следующий вид:

```py
{
    ('I', 't'): 1, ('t', "'"): 1, ("'", 's'): 1, ('s', '</s>'): 1,
    ('f', 'a'): 3, ('a', 'r'): 3, ('r', ','): 3, (',', '</s>'): 4,
    ('r', 't'): 2, ('t', 'h'): 2, ('h', 'e'): 2, ('e', 'r'): 2,
    ('e', 's'): 2, ('s', 't'): 2, ('t', '</s>'): 2,
    ('a', 'n'): 1, ('n', 'd'): 1, ('d', '</s>'): 1,
    ('o', 'l'): 3, ('l', 'd'): 3, ('d', ','): 1,
    ('d', 'e'): 2
}
```

Обратите внимание, что сочетания токенов на стыке слов не образуют пару. Так, в нашем словаре пар
нет сочетаний, начинающихся с токена конца слова.
Также обратите внимание, что для пары порядок токенов критичен.

Функция формирования частотного словаря пар токенов принимает на вход частотный словарь слов.
Функция возвращает частотный словарь пар токенов.

В случае, если на вход приходит значение, не являющееся словарем, функция возвращает `None`.

Интерфейс:

```py
def count_tokens_pairs(word_frequencies: dict[tuple[str, ...], int]) -> (
        dict[tuple[str, str], int] | None):
    pass
```

### Шаг 4. Формирование нового токена

Словарь частотности сочетаний токенов позволяет нам выбрать, из чего можно сформировать новый
токен.

Необходимо реализовать функцию, которая обновляет частотный словарь слов, заменяя в ключах
пары токенов, из которых сформирован новый токен, на этот самый новый объединенный токен.

Так, если в частотном словаре слов из предыдущего шага объединить пару токенов `(',', '</s>')` в
новый токен, то словарь должен измениться следующим образом:

```py
{
    ('I', 't', "'", 's', '</s>'): 1,
    ('f', 'a', 'r', ',</s>'): 1,
    ('f', 'a', 'r', 't', 'h', 'e', 'r', ',</s>'): 1,
    ('f', 'a', 'r', 't', 'h', 'e', 's', 't', '</s>'): 1,
    ('a', 'n', 'd', '</s>'): 1,
    ('o', 'l', 'd', ',</s>'): 1,
    ('o', 'l', 'd', 'e', 'r', ',</s>'): 1,
    ('o', 'l', 'd', 'e', 's', 't', '</s>'): 1
}
```

Функция принимает на вход частотный словарь, ключами которого выступают слова,
а значениями - их частота в тексте  и возвращает частотный словарь с измененными ключами.

В случае, если на вход приходят аргументы не того типа, который ожидается (то есть не словарь
и не кортеж), функция возвращает `None`.

Интерфейс:

```py
def merge_tokens(
        word_frequencies: dict[tuple[str, ...], int],
        pair: tuple[str, str]
) -> dict[tuple[str, ...], int] | None:
    pass
```

### Шаг 5. Обучение токенизатора

### Выполнение Шага 5 соответствует 6 баллам

Теперь у нас есть все компоненты для того, чтобы обучить наш токенизатор и сформировать
необходимое количество новых токенов.

Напишите функцию, которая принимает на вход частотный словарь предобработанных слов и количество
необходимых слияний. Функция формирует столько новых токенов, сколько требуется слияний,
и возвращает обновленный словарь частотности слов.

Для формирования нового токена необходимо выбирать самую часто встречающуюся пару токенов.
В случае, если несколько пар встречаются одинаково часто, необходимо выбрать ту, которая даст более
длинный токен. Если и это не даст однозначного ответа, то необходимо выбрать ту пару, которая дает
лексикографически меньший токен.

Например, имея пары `{('a', 'b'): 3, ('b', 'cd'): 3, ('b', 'ca'): 3}` необходимо сформировать
токен `'bca'`, так этот токен длиннее токена `'ab'` и лексикографически меньше токена `'bcd'`.

В случае, если функция принимает на вход значения не тех типов, которые ожидаются (то есть
не словарь и не целочисленное значение), возвращается `None`. То же значение должно возвращаться,
если любая из реализованных вами функций, используемых в `train`, вернула `None`.

Обратите внимание, что если доступных для слияния пар токенов не осталось, то обучение должно
прекратиться даже если необходимое количество токенов не было достигнуто.

Интерфейс:

```py
def train(
        word_frequencies: dict[tuple[str, ...], int] | None,
        num_merges: int
) -> dict[tuple[str, ...], int] | None:
    pass
```

Продемонстрируйте обучение токенизатора на материале текста из переменной `text`
в модуле `start.py`. Используйте 100 слияний в качестве критерия остановки обучения.

> NOTE: Подумайте, можно ли установить связь между оптимальным количеством слияний и количеством
> вхождений в частотный словарь слов?

### Шаг 6. Присвоение идентификаторов токенам

Известно, что формальные модели, включая лингвистические модели, не способны обрабатывать буквенные
данные, поэтому необходимо для каждой текстовой последовательности сформировать числовой вектор.

Для этого необходимо присвоить каждому из получившихся токенов и
символов в них определенный числовой идентификатор. Обратите внимание, что при присваивании
идентификатора не нужно учитывать повторяющиеся токены и символы. Отсортируйте токены
по длине в порядке убывания. Токены, имеющие одинаковую длину, должны быть отсортированы
лексикографически в порядке возрастания. Присвойте данной последовательности идентификаторы
от 0 до n-1, где n - общее количество оригинальных токенов и символов внутри токенов.

Например, из набора токенов `['far', 'old', 'th', 'er', 'est', '</s>', 'a', 'n', 'd']`
должен получиться следующий словарь:

```py
{'<unk>': 0, 'est': 1, 'far': 2, 'old': 3, '</s>': 4, 'er': 5, 'th': 6, 'a': 7, 'd': 8, 'e': 9,
 'f': 10, 'h': 11, 'l': 12, 'n': 13, 'o': 14, 'r': 15, 's': 16, 't': 17}
```

Реализуйте функцию, которая принимает на вход частотный словарь с ключами -
токенизированными словами, значениями – их частотами в тексте, а также токен,
обозначающий неизвестную последовательность.
Функция возвращает словарь вида `<токен: числовой идентификатор>`.
В возвращаемом словаре должны присутствовать специальные токены: токен конца слова
(при наличии), токен начала слова (при наличии)
и неизвестный токен. Присвоение идентификатора данным токенам производится так же после сортировки
вместе с остальными токенами.

В случае, если функция принимает на вход значения не тех типов, которые ожидаются, то возвращается `None`.

Интерфейс:

```py
def get_vocabulary(
        word_frequencies: dict[tuple[str, ...], int],
        unknown_token: str
) -> dict[str, int] | None:
    pass
```

### Шаг 7. Декодирование текста

### Выполнение Шага 7 соответствует 8 баллам

Теперь, когда у нас есть выделенные токены и присвоенные им числовые идентификаторы,
мы можем декодировать любую последовательность.

Для этого необходимо реализовать функцию, принимающую на вход закодированный текст в виде
списка чисел, словарь соответствия токенов числовым идентификаторам и токен, обозначающий конец
слова. Функция возвращает строку. Обратите внимание, что в возвращаемой строке не должно быть
токенов, обозначающих конец слова! Вместо них должно быть то, что мы в рамках текущей работы
считаем границей слова, то есть пробельный символ.

Так, при использовании словаря из предыдущего шага, последовательность `[2, 6, 1, 4]`
должна быть раскодирована как `'farthest '`.

В случае, если на вход функции поступают аргументы неожиданного типа, функция должна вернуть `None`.
Обратите внимание, что токен конца слова может быть представлен как `None`.

```py
def decode(encoded_text: list[int], vocabulary: dict, end_of_word_token: str | None) -> str | None:
    pass
```

Создайте словарь вида `<токен: числовой идентификатор>`, используя `'<unk>'` в качестве токена,
обозначающего неизвестную последовательность. Затем продемонстрируйте работу декодера, прочитав и
декодировав секретное содержимое любого файла из папки [./assets/secrets](assets/secrets) в модуле `start.py`.

Обратите внимание, что данные файлы содержат инструкцию о получении бонуса к оценке.
**Количество бонусов ограничено**: только первые студенты, справившиеся с заданием, смогут его получить.
Один студент может отгадать не более одной загадки, таким образом, бонус смогут получить
только 5 студентов.
Решение о применении бонуса принимается ментором и не подлежит оспариванию.
Применение бонуса возможно только в случае, если на момент выполнения задания в форке студента
опубликован актуальный код, позволяющий воспроизвести результат.

### Шаг 8. Кодирование одного слова

Кодирование текста, в отличие от декодирования, является более сложным процессом и подразумевает
предварительную токенизацию текста. На текущем шаге необходимо реализовать функцию, которая
принимает в качестве аргументов предобработанное слово в виде кортежа строк, словарь
соответствий токенов идентификаторам, токен конца слова и неизвестный токен.
Функция возвращает список закодированных токенов.

В случае, если функция принимает на вход значения не тех типов, которые ожидаются, то возвращается `None`.

Обратите внимание, что при токенизации слова необходимо в первую очередь выбирать более длинные
токены.

Например, токенизируем слово `('w', 'o', 'r', 'd', '</s>')`.

Допустим, мы имеем следующий словарь токенов:

```py
{
    'w': 0, '<unk>': 1, 'o': 2, 'r': 3, 'd': 4, 'wo': 5, 'ord': 6, '</s>': 7
}
```

При корректной реализации слово должно быть токенизировано следующим образом: `[0, 6, 7]`.
Варианты `[5, 3, 4]` и `[0, 2, 3, 4]` являются неправильными: токен `ord` длиннее токенов `wo` и
тем более `w`, `o`, `r` и `d`.

В случае, если есть несколько подходящих самых длинных токенов, следует использовать
тот токен, который меньше лексикографически. В случае, если встречается последовательность, которую
нельзя заменить ни на один из известных токенов, она заменяется на специальный неизвестный токен.
Это позволит без ошибок обрабатывать тексты, включающие в себя, например, вставки на других языках.

Интерфейс:

```py
def tokenize_word(
        word: tuple[str, ...],
        vocabulary: dict[str, int],
        end_of_word: str | None,
        unknown_token: str
) -> list[int] | None:
    pass
```

### Шаг 9. Загрузка словаря

В дальнейшем нам предстоит работать с нейросетевой языковой моделью, обученной на
задачу перевода с русского языка на английский.
Для этого необходимо использовать специальный словарь токенов, обученный на большом объеме текстов,
расположенный в файле [`./assets/vocab.json`](assets/vocab.json).

Таким образом, необходимо научиться считывать такие словари из файлов с расширением `.json`.

Мы уже встречались с файлами такого формата в предыдущих лабораторных работах. Больше узнать об
этом формате можно [здесь](https://ru.wikipedia.org/wiki/JSON).

Для работы с такого типа файлами используется библиотека
[`json`](https://pythonworld.ru/moduli/modul-json.html).

Реализуйте функцию, которая принимает на вход путь к файлу `json` с обученным словарем,
загружает этот словарь и возвращает его. В случае, если загруженный объект не является
словарем, функция возвращает `None`.

В случае, если функция принимает на вход значение не того типа (то есть не строку), то возвращается `None`.

Интерфейс:

```py
def load_vocabulary(vocab_path: str) -> dict[str, int] | None:
    pass
```

### Шаг 10. Кодирование текста

Для того чтобы формальная модель поняла наш запрос, необходимо его закодировать.

Реализуйте функцию, которая принимает на вход текст в виде строки, словарь токенов,
токен конца слова, токен начала слова и неизвестный токен. Функция должна возвращать список
идентификаторов токенов.
В случае, если функции были переданы аргументы не того типа, который ожидается, функция должна
вернуть `None`.

Функция обязательно должна вызывать функции `prepare_word` и `tokenize_word`.
В случае, если какая-либо из этих функций возвращает значение `None`, функция `encode` так же
должна вернуть `None`.

Интерфейс:

```py
def encode(original_text: str, vocabulary: dict, start_of_word_token: str | None,
           end_of_word_token: str | None, unknown_token: str) \
        -> list[int] | None:
    pass
```

### Шаг 11. Выделение N-грамм

Теперь мы готовы перейти к тому, чтобы научиться оценивать качество перевода.
Для сравнения полученных предсказаний с истинным ответом, необходимо научиться рассчитывать метрику
[BLEU (bilingual evaluation understudy)](https://en.wikipedia.org/wiki/BLEU).

Эта метрика часто используется для оценки предсказаний в задаче машинного перевода текста.
Она представляет собой геометрическое среднее метрик
[Precision](https://en.wikipedia.org/wiki/Precision_and_recall), посчитанных для n-грамм
различного порядка.

В течение следующих нескольких шагов мы реализуем подсчет метрики BLEU.

Для начала необходимо реализовать функцию для выделения n-грамм из последовательности.
N-граммами называют такую подпоследовательность, которая включает в себя ровно n последовательных
элементов.
Например, из буквенной последовательности `мыть` можно выделить четыре униграммы (`м`, `ы`, `т`,
`ь`), три биграммы (`мы`, `ыт`, `ть`), две триграммы (`мыт`, `ыть`) и,
наконец, одну 4-грамму (`мыть`).

Реализуйте функцию, которая принимает на вход последовательность в виде строки и число,
обозначающее порядок n-граммы (то есть n). Функция возвращает список кортежей, каждый
из которых состоит из строк. Каждая n-грамма - это кортеж.

Например, при вызове функции с последовательностью `'мыть'` и порядком 3, мы ожидаем получить
результат `[('м', 'ы', 'т'), ('ы', 'т', 'ь')]`.

В случае, если функции были переданы аргументы не того типа,
который ожидается (то есть не строка и не число), функция должна вернуть `None`.

Интерфейс:

```py
def collect_ngrams(text: str, order: int) -> list[tuple[str, ...]] | None:
    pass
```

### Шаг 12. Вычисление метрики Precision

В общем случае значение метрики Precision обозначает, сколько положительных предсказаний модели
оказались действительно положительными. В нашем случае можно сформулировать это более конкретно:
какая доля истинных значений попала в предсказания модели.

Иными словами, чем больше предсказанных n-грамм действительно присутствуют в истинной
последовательности, тем выше значение Precision.

Рассмотрим пример. Пусть истинными значениями является `[('м',), ('ы',), ('т',), ('ь',)]`,
а предсказанными - `[('м',), ('ы',), ('т',), ('ь',), ('c',), ('я',)]`.
Тогда совпадающими значениями являются следующие 4 элемента: `[('м',), ('ы',), ('т',), ('ь',)]`.
Так как всего нам дано 6 предсказанных значений Precision, то доля совпадающих элементов,
равна $\frac{4}{6}=0.(6)$.

Реализуйте функцию, принимающую на вход два списка кортежей со строками. Первый список является
истинными n-граммами, второй - предсказанными. Обратите внимание, что при подсчете метрики не
нужно учитывать дублирующиеся кортежи со строками. Функция возвращает значение метрики. В случае, если
список истинных значений пуст, возвращается значение метрики равное 0. При получении некорректных
аргументов (то есть не списков), должно вернуться `None`.

В случае, если функции были переданы аргументы не того типа,
который ожидается, то функция должна вернуть `None`.

Интерфейс:

```py
def calculate_precision(actual: list[tuple[str, ...]],
                        reference: list[tuple[str, ...]]) -> float | None:
    pass
```

### Шаг 13. Вычисление среднего геометрического

До расчета метрики BLEU нам остался всего один шаг, и это подсчет среднего [геометрического
значения](https://ru.wikipedia.org/wiki/Среднее_геометрическое).

Рассчитывать это среднее значение мы будем для метрик Precision, посчитанных для n-грамм
различного порядка от 1 до $order_{max}$, где $order_{max}$ - максимальный порядок рассматриваемых
n-грамм.

Рассчитать значение можно по следующей формуле:

$$Mean_{geometric} = exp(\frac{1}{order_{max}} \times \sum_{i=1}^{order_{max}}ln(Precision_{i}))$$

Здесь $Precision_{i}$ обозначает значение метрики Precision для n-грамм порядка $i$, причем $i$
принимает значения от 1 до $order_max$.

Реализуйте функцию, принимающую на вход список значений метрики Precision и максимальный порядок
n-грамм. Функция возвращает значение геометрического среднего. Обратите внимание, что если хотя бы
одно из значений Precision, пришедших на вход, является неположительным, то функция должна
вернуть 0. При получении аргументов, чей тип расходится с ожидаемым, функция должна вернуть `None`.

```py
def geo_mean(precisions: list[float], max_order: int) -> float | None:
    pass
```

### Шаг 14. Вычисление метрики BLEU

### Выполнение Шага 14 соответствует 10 баллам

Наконец, мы готовы рассчитать метрику BLEU, отражающую близость предсказанного значения истинному.

Реализуйте функцию, которая принимает на вход истинное значение в виде строки,
предсказанное значение в виде строки, а также максимальный порядок рассматриваемых n-грамм.
Функция возвращает значение метрики BLEU.

Функция должна выделить n-граммы из предоставленных последовательностей всех порядков от 1 до
обозначенного максимального, посчитать Precision для выделенных n-грамм каждого порядка,
вычислить среднее геометрического полученных значений метрики и вернуть это значение,
предварительно умножив на 100.

Функция обязательно должна вызывать `collect_ngrams`, `calculate_precision` и `geo_mean`.
В случае, если какая-либо из этих функций возвращает `None`, `calculate_bleu` так же должна вернуть
`None`. То же касается случаев, когда тип принятых на вход аргументов расходится с ожидаемым.

Интерфейс:

```py
def calculate_bleu(
        actual: str | None, reference: str, max_order: int = 3
) -> float | None:
    pass
```

Продемонстрируйте оценку работы языковой модели `Helsinki-NLP/opus-mt-ru-en`
в задаче машинного перевода в `start.py`. Информацию об этой языковой модели можно получить
[здесь](https://huggingface.co/Helsinki-NLP/opus-mt-ru-en).
Пример работы с данной языковой моделью вы можете увидеть
[здесь](assets/nmt_demo/main.py).

Вам не нужно импортировать модель и генерировать предсказания, это сделано за вас.
Текст, использованный для получения предсказания, расположен в файле
[`./assets/for_translation_ru_raw.txt`](assets/for_translation_ru_raw.txt).
Закодируйте его при помощи
специального [обученного словаря](assets/vocab.json). При кодировании в качестве
токена начала слова следует использовать `'\u2581'`, в качестве неизвестного
токена необходимо использовать `'<unk>'`, токен конца слова использовать не нужно
(т.е. необходимо передать `None`).
Чтобы убедиться, что файл удалось закодировать
верным образом, можно сравниться с закодированной версией текста из файла
[`./assets/for_translation_ru_encoded.txt`](assets/for_translation_ru_encoded.txt). Закодированные
предсказания модели (то есть предсказания в том виде, в котором их вернула модель) можно найти
в файле [`./assets/for_translation_en_encoded.txt`](assets/for_translation_en_encoded.txt).
Необходимо декодировать их, используя реализованные вами функции, и сравнить получившийся текст
с эталонным переводом из файла
[`./assets/for_translation_en_raw.txt`](assets/for_translation_en_raw.txt).
При декодировании токен конца слова следует передать как `None`.
Сравнение следует производить по метрике BLEU. После декодирования и перед сравнением необходимо
очистить текст от токена начала слова (`'\u2581'`), заменив его на пробел.

## Полезные ссылки

* [Оригинальная статья о BPE алгоритме](
http://www.pennelynn.com/Documents/CUJ/HTML/94HTML/19940045.HTM)
* [Описание формата](https://ru.wikipedia.org/wiki/JSON) хранения данных JSON и
[документация](https://pythonworld.ru/moduli/modul-json.html)
библиотеки для работы с такими файлами
* [Описание](https://huggingface.co/Helsinki-NLP/opus-mt-ru-en)
нейросетевой модели, чьи предсказания были использованы в настоящей работе
* [Оригинальная статья о BLEU метрике](https://aclanthology.org/P02-1040.pdf),
используемой для оценки качества машинного перевода
