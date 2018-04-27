# Решение ML-трека Яндекс.Алгоритма (3-е место)

__Результат (private): 86513__

* [Описание задачи](https://contest.yandex.ru/algorithm2018/contest/7914/problems/)
* [Приватый лидерборд](https://contest.yandex.ru/algorithm2018/contest/7914/standings/)

## Краткое Описание

Решение представляет из себе простое усреднение двух независимых решений. Как показал прайват лидерборд на призовое место хватило бы любого из них. В первой модели фичи проще, модель прозрачнее, а результат лучше, поэтому начнём с неё.

1. Решение 1 – Старый добрый градиентный бустинг. Для получения скора для ранжирования строятся два бинарных классификатора. Первый предсказывает вероятность класса good, второй вероятность класса bad. Скор для ранжирвоания считается как P(good) - P(bad). __Результат модели (private): 86363__

2. Решение 2 - Нейронная сеть. Совмещение небольшой свёртки над эмбеддингами слов + обычная fully-connected сеть над разными фичами. Скор для ранжирования считаем как P(good) - P(bad) __Результат модели (private): 86288__

Тексты подвергались минимальной предобработке. Никакие данные из датасета не удалялись и не добавлялись.

Финальный скор для ранжирования = (Скор модели 1) + 0.5 * (Скор модели 2).

## Как читать

Репозиторий состоит из трёх тетрадок. 1 и 2 тренируют соответствующие модели, 3 объединяет решения. Чтобы воспроизвести результаты потребуется скачать немного дополнительных данных.

## Что нужно

Нужны будут данные:

* [fastText эмбеддинги обученные на wiki](https://fasttext.cc/docs/en/pretrained-vectors.html)
* [fastText эмбеддинги обученные на Common Crawler](https://fasttext.cc/docs/en/crawl-vectors.html)
* [OpenSubtitles2018 en-ru](http://opus.nlpl.eu/download.php\?f\=OpenSubtitles2018/en-ru.txt.zip) (для обучения своих fasttext эмбеддингов)

И библиотеки (из нетривиального):

* [fastText v0.1.0](https://github.com/facebookresearch/fastText/releases/tag/v0.1.0)
* Keras==2.1.5
* tensorflow==1.7.0
* pymorphy2==0.8
* pymorphy2_dicts==2.4.393442.3710985

Я учил на MacOS 10.12.6 и Python 3.6.5, но должно быть более-менее совместимо.

## Подробнее про Решение 1

Тексты здесь почти не обрабатывал. Слова привёл к нормальным формам (`pymorphy2`) только ради парочки фичей. В основном работал as is, оставлял пунктуацию, странные символы и вот это всё.

Это два бинарных классификатора с одинаковыми и довольно стандартными гиперпараметрами: `LGBMClassifier(n_estimators=2000, learning_rate=0.025, colsample_bytree=0.3)`, кроме `colsample_bytree`, который довольно низкий, так как среди фичей есть tf-idf разложение и по нему очень легко переобучиться. Всего 7188 фичей. Использовал `confidence` в качестве веса обучающих примеров.

Почему два бинарных классификатора, а не 1 мультикласс? 3 причины:

1. (Самое очевидное) так работает лучше.
2. Так в модель вносится информация об упорядоченности классов.
3. Как оказалось, обучить две бинарные модели быстрее, чем одну на 3 класса. (На моём ноутбуке ~15 минут + ~15 минут против ~45минут)

Другие бустинги кроме `lightgbm` пробовал совсем чуть-чуть. В основном из-за скорости, на ноуте на разреженных матрицах учиться и так не в радость.

Описание признаков (везде под `context` понимается `context_0 + ' ' + context_1 + ' ' + context_2`)

* 6001 признак из `TfidfVectorizer(ngram_range=(1, 3), analyzer='char', max_features=2000)`
  * Представление reply
  * Представление context_0
  * Их поэлементное произведение
  * Построчная сумма поэлементного произведения (это почти косинус между представлениями и пространстве tf-idf признаков).
* 18 признаков разнообразных счётчиков слов:
  * Количество слов в reply, context_0, context, пересечении reply и context_0, пересечении reply и context
  * reply == to_context_0
  * reply is in context_0, reply is in context (полностью как подстрока)
  * Сумма idf-весов слов в reply, context_0, context, нормализованном reply, нормализованном context, пересечении reply и context_0, пересечении reply и context
* 9 признаков – расстояний между текстами в пространствах разных tf-idf vectorizer'ов (как пункт 4 в блоке 1)
  * Три разных векторайзера:
    * `TfidfVectorizer(ngram_range=(1, 3), analyzer='char', max_features=2000)`
    * `TfidfVectorizer(ngram_range=(1, 2), analyzer='word', max_features=2000)`
    * `TfidfVectorizer(ngram_range=(1, 2), analyzer='char', use_idf=False)`
  * Два текста трансформируются векторайзерами, представления поэлементно умножаются и суммируются.
  * Использовал расстояния между парами reply/context_0, reply/context_1, reply/context_2
* 300 + 300 признаков из wiki-эмбеллингов fasttext:
  * Просто сумма векторов слов в reply и context_0
* 300 признаков из CC-эмбеллингов fasttext:
  * fasttext-овский метод `.get_sentence_vector`
* 20 признаков SVD разложения ещё пары векторайзеров
* 120 + 120 признаков граммемы из `pymorphy2`:
  * Каждое слово в reply мапалось в теги. Дальше считал долю каждого тега в строке. Всего `pymorphy2` знает 120 тегов.
  * По аналогии для context_0

## Подробнее про Решение 2

Одна нейронная сеть из двух частей - небольшая свёртка не эмбеддингах слов и пара простых fully-connected на разных фичах (многие как в решение 1). Из интересного - тренировал свои эмбеддинги на OpenSubtitles, в надежде, что они будут лучше. Получилось так же. Сеть – простой классификатор на три класса, ранжирование по P(good) - P(bad). Использовал `confidence` в качестве веса обучающих примеров.

Тут текст немного приводил в порядок. Убирал пунктуацию, букву ё.

Фичи:
* Вектора (свои на OpenSubtitles) 5 самых важных слов (по idf) в порядке их появления в предложении.
  * Для reply, context_0, context_1, context_2
* Вектора предложений:
  * Для reply, context_0, context_1, context_2, пересечения reply/context_0, reply/context_1, reply/context_2
* Теги из `pymorphy2`.
  * Всё как в решении 1, но нормализовал не на количество тегов, а на количество слов. Значимой разницы нет.
* Скалярное проивзедение векторов context_0, context_1, context_2 и reply
* Косинусы между reply/context_0, reply/context_1, reply/context_2, reply/(context_0 + context_1 + context_2)
* Сумма idf весов в reply, context_0, context_1, context_2
* Количество символов в reply, context_0, context_1, context_2, пересечениях reply/context_0, reply/context_1, reply/context_2
