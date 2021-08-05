# Алгоритм Света

Алгоритм Света - это набор программ, позволяющих по тексту судебного приговора, связанного с убийством женщины, определить, погибла ли она от рук партнера или родственника, и если да, то кто именно был убийцей. Проект стал победителем хакатона [Прожектор 2021](https://projector2021.te-st.ru/), после чего был доработан совместно с [Консорциумом женских неправительственных объединений](https://wcons.net/en/) и волонтерами при поддержке [Теплицы социальных технологий](https://te-st.ru/).

Прочитать о проекте подробнее можно здесь:
- на сайте проекта
- в материалах Теплицы
- [в материалах Новой газеты](https://novayagazeta.ru/articles/2021/02/16/89254-pivo-revnost-taburet)

Алгоритм включает в себя:
- набор парсеров, скачивающих доступные тексты приговоров по заданной статье за заданный год
- набор регулярных выражений, позволяющих выделить приговоры, в которых речь идет об убийстве женщины
- код для предобработки текстовых данных и векторайзер для подготовки матрицы документы-токены
- код для обучения итоговой модели, фрагмент данных для обучения
- итоговые модели в виде бинарных файлов, которые можно напрямую применять к любым предобработанным данным для получения прогноза

## Содержание  
[1. Парсеры](#parsers)  
[2. Регулярные выражения](#regex)   
[3. Предобработка текстов](#preprocessing)   
[4. Обучение модели](#learning)   
[5. Применение модели](#prediction)   

<a name="parsers"/>

## 1. Парсеры

Тексты приговоров для нашего проекта собирались с двух ресурсов: сайта Мосгорсуда и ГАС Правосудие. На первом публикуют приговоры, вынесенные в судах Москвы, на втором - приговоры всех остальных судов страны, кроме московских. Технически принципы сбора данных из этих двух источников сильно различаются.

__Парсер ГАС Правосудие__ состоит из двух частей: вначале посредством `Selenium WebDriver` собираются ссылки на запрашиваемые дела, из ссылок извлекаются заключенные в них идентификаторы дел, затем по идентификаторам собираются тексты с помощью пакета `requests`. Этот парсер работает на любых операционных системах. См. подробнее в файлах: `Parsers_Pravosudie_Links.ipynb` и `Parsers_Pravosudie_Texts.ipynb`

__Парсер Мосгорсуда__ также работает в несколько этапов: вначале пакетом `requests` собирается общая информация о делах и ссылки на их отдельные страницы, затем по этим адресам находятся ссылки на файлы с текстами приговоров, они скачиваются и прочитываются. К сожалению, этот парсер работает только на Windows, т.к. на последнем этапе тексты приговоров извлекаются из файлов Word с помощью пакета `win32com`. Предложения по универсализации этого этапа сильно приветствуются :) См. подробнее в файле: `Parsers_Mosgorsud.ipynb`

Мы собирали тексты приговоров по статьям 105, 107, 111 ч.4 УК РФ. Теоретически опубликованные парсеры адаптированы к сбору текстов любых статей при соблюдении формата запросов.

<a name="regex"/>

## 2. Регулярные выражения

Регулярные выражения - это специальным образом составленные правила, по которым можно найти интересующие последовательности символов, слов и выражений независимо от их форм. Мы использовали регулярные выражения, чтобы автоматически найти дела, в которых погибшими являлись женщины. Семь выражений, которые мы оставили по итогам выборочной ручной проверки, и способ их применения к собранным данным представлены в коде ниже.


```python
import re
import pandas as pd

patterns = [
    r'скончалась',
    r'погибш[аеу][йяю]',
    r'смерт[иь]ю? потерпевшей',
    r'смерт[иь]ю? последней',
    r'мертва',
    r'е[её] труп',
    r'труп женщины'
]

def check_patterns(text):
    global patterns
    
    for pattern in patterns:
        if re.search(pattern, text):
            return True
    return False

data = pd.read_csv('Ваш_файл_с_собранными_приговорами.csv')
data = data[(data['ID'].notna())&(data['text'].notna())].set_index('ID')
women_cases = data[data['text'].apply(check_patterns)]
women_cases.to_csv('Ваш_файл_с_женскими_делами.csv')
```

В датафрейме `women_cases` будут находиться отобранные "женские" дела.

<a name="preprocessing"/>

## 3. Предобработка текстов

Предобработка текстов для обучения, тестирования и применения моделей включает в себя три этапа: токенизацию (разбиение исходных текстов на отдельные единицы анализа), лемматизацию (приведение слов к начальной форме) и удаление стоп-слов, т.е. слов, не несущих смысловой нагрузки, но при этом часто встречающихся. При использовании кода ниже имейте в виду, что пакет `pymystem3` отлично работает на Mac OS, но крайне долго - на Windows. Для Windows я рекомендую лемматизировать слова с помощью пакета `pymorphy2`. Процедура будет слегка отличаться, но код для нее реально найти в интернете (либо можно обратиться ко мне).

Для работы с моделями предобработанные тексты также проходят векторизацию, т.е. представляются в виде матрицы документы-токены. В нашем исследовании токенами, т.е. единицами анализа, выступали униграммы (отдельные слова) и биграммы (сочетания из двух стоящих рядом слов), которые встречаются не меньше чем в 5% документов. Имея базу предобработанных текстов, вы можете получить описанную матрицу с помощью опубликованного файла `cvect.pkl`.

```python
import re
import pymystem3
import stop_words
import pandas as pd 

mstem = pymystem3.Mystem()

def keep_only_rus(text):    
    new_text = ''
    for symbol in text:
        if re.match(r'[А-я]', symbol) or symbol == ' ':
            new_text += symbol
        else:
            new_text += ' '
    return new_text

def del_double_spaces(text_with_double_spaces):
    while '  ' in text_with_double_spaces:
        text_with_double_spaces = text_with_double_spaces.replace('  ',' ')
    return text_with_double_spaces

def lemmatize(raw_text):
    return ''.join(mstem.lemmatize(raw_text)).strip()

stopwords = stop_words.get_stop_words('russian')
stopwords.extend(stop_words.get_stop_words('english'))
stopwords = list(set(stopwords))
stopwords += [
    'фио', 'гггг', 'подсудимый', 'суд',
    'изымать', 'согласно', 'наказание',
    'потерпевший', 'показание', 'судебный',
    'преступление', 'адрес', 'свидетель',
    'свой', 'находиться', 'час', 'ход'
             ]

def del_stopwords(text):
    global stopwords
    new_text = []
    for word in text.split():
        if word not in stopwords and len(word) > 2:
            new_text.append(word)
    return ' '.join(new_text)
    
data = pd.read_csv('Ваш_файл_с_женскими_делами.csv', index_col=0) # в индексах желательно держать ID дел
data['text_prep'] = data['text'].str.lower()
data['text_prep'] = data['text_prep'].apply(keep_only_rus)
data['text_prep'] = data['text_prep'].apply(del_double_spaces)
data['text_prep'] = data['text_prep'].apply(lemmatize)
data['text_prep'] = data['text_prep'].apply(del_stopwords)
data['text_prep'] = data['text_prep'].apply(del_double_spaces)
data = data[data['text_prep'].notna()]
data.to_csv('Ваш_файл_с_женскими_делами_после_предобработки.csv')
```

В датафрейме `data` будут находиться отобранные "женские" дела после предобработки.

<a name="learning"/>

## 4. Обучение моделей

Обе модели, полученные по результатам работы команды, основаны на традиционном методе машинного обучения - градиентном бустинге, показавшем наивысшее качество прогноза. Целевой метрикой, которую мы максимизировали в ходе отбора моделей и на которую мы ориентировались, была precision для категории 1. Для первой модели это доля дел, помеченных алгоритмом как связанные с домашним насилием, которые действительно таковыми являются (согласно ручной разметке). Итоговый показатель этой метрики для первой модели составил 0.86. Для второй модели это доля дел, помеченных алгоритмом как дела с убийцей-партнером, которые действительно таковыми являются (согласно ручной разметке). Итоговый показатель этой метрики для первой модели составил 0.94.

Воспроизвести обучение моделей можно с помощью кода из файла `Analysis_models.ipynb` и базы размеченных данных `preprocessed_data_2018.xlsx`. Представлены только итоговые версии используемых моделей. Попытки повысить качество моделей также приветствуются.

<a name="prediction"/>

## 5. Применение моделей

Применить разработанные модели можно напрямую, не обучая их. Для этого нужен лишь датафрейм с предобработанными согласно [п. 3](#preprocessing) текстами приговоров и опубликованные бинарные файлы: `cvect.pkl` (векторайзер), `gbc_dv.pkl` (модель, предсказывающая, была ли женщина в целом убита партнером или родственником), `gbc_ipv.pkl` (модель, предсказывающая, была ли женщина убита именно партнером, а не родственником).

```python
import pickle
import pandas as pd

with open("cvect.pkl", 'rb') as file:
    cvect = pickle.load(file)
    
with open("gbc_dv.pkl", 'rb') as file:
    dv = pickle.load(file)
    
with open("gbc_ipv.pkl", 'rb') as file:
    ipv = pickle.load(file)
    
data = pd.read_csv('Ваш_файл_с_женскими_делами_после_предобработки.csv', index_col=0) # в индексах желательно держать ID дел
matrix = cvect.transform(data['text_prep'])
td_matrix = pd.DataFrame(matrix.toarray(), index=data.index, columns=cvect.get_feature_names())
td_matrix['DV'] = dv.predict(td_matrix)
dv_preds = td_matrix[['DV']]
td_matrix_ipv = td_matrix[td_matrix['DV']==1].drop('DV', axis=1)
td_matrix_ipv['IPV'] = ipv.predict(td_matrix_ipv)
dv_preds['IPV'] = td_matrix_ipv['IPV']
```

В датафрейме `dv_preds` для каждого дела будет сделан прогноз: столбик `DV` (1 - женщина была убита партнером или родственником, 0 - иначе) и столбик `IPV` (1 - женщина была убита партнером, 0 - родственником, NaN - иначе). 

> **⚠ ВНИМАНИЕ!**  
> Алгоритм Света предназначен только для анализа дел, в которых речь идет о погибших женщинах. Модели не будут давать корректных прогнозов для дел с жертвами-мужчинами, т.к. такие дела не были включены в обучающую выборку. 
