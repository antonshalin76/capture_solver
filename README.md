# Capture solver

## Постановка задачи
Решаем задачу поиска целевого предмета среди множества изображений по предложенному образцу.
Каждая капча - это группа изображений, среди которых необходимо выбрать правильный ответ, и картинка-подсказка, в которой зашифрован ответ. На каждом из вариантов ответа изображены 3D объекты, расположенные на конвейерах в два ряда. Количество объектов вариьруется от 4 до 8 штук. Правильным ответом является изображение, на котором стрелка указывает на объект на конвейере, совпадающий с объектом, изображенным на картинке-подсказке.

В примере ниже на подсказке (изображение в нижнем ряду) изображена подставка для яйца, поэтому нужно выбрать седьмое изображение (в верхнем ряду), где стрелка указывает на тот же объект. Объекты на конвейерах расположены в случайном порядке и случайной ориентации.
![image](https://github.com/antonshalin76/capture_solver/assets/50358174/8bfd9813-23e3-4ba0-ab9c-4340246cdc3b)

Для поиска решения на входе имеем два набора. 

Первый набор состоит из двух директорий: images_100 и labels_100. Всего в них находятся 100 капч с разметкой баундинг боксов для всех объектов (label=object) на изображении, включая стрелку (label=icon) и ответ (label=true_object).
![image](https://github.com/antonshalin76/capture_solver/assets/50358174/52e9992f-45a2-40d7-acc8-6cba340fa6a7)

Второй набор также из аналогичных двух директорий: images_train и labels_train. В них находятся 3998 капч с разметкой баундинг бокса только для ответа (label=true_object).
![image](https://github.com/antonshalin76/capture_solver/assets/50358174/c32a762a-d9b7-4cd2-b258-24699a5b2635)

Разметка не имеет классификатора, т.е. все метки являются одинаковыми object без указания что это за предмет.
Разметка в формате <a href="https://github.com/labelmeai/labelme">labelme</a>.

Необходимо создать модель, которая для каждой такой капчи выбирает правильный ответ. Нужно предсказать координату точки, соответствующей ответу. Целевая метрика - точность предсказанных ответов. Точность следует считать следующим образом: если предсказанная точка лежит внутри того же патча, что и иконка на правильной орбите, то предсказание является истиной. Тогда отношение количества истинных предсказаний ко всем будет являться точностью.

## Решение
Для начала разберемся с предметами, представленными на капчах. Поскольку в исходной разметке все предметы обезличены, то сперва необходимо их переразметить с классификацией, чтобы можно было обучать модель различать и искать нужные предметы.
### _Шаг 1._
- Из всех имеющихся изображений обоих наборов вырезаем все предметы по контуру, обозначенному в json файлах в качестве bbox разметки.
- Подаем их на отдельную утилиту, которая по структурному и цветовому хэшам ищет среди всего набора похожие между собой изображения и раскладывает их по соответсвующим папкам.

Теперь у нас есть классифицированный датасет из 4265 картинок 15 предметов, которые имеют значительную разбалансировку - от 53 до 660 картинок на предмет.

Для выработки пути решения отметим ключевые моменты:
1. Картинка-подсказка расположена на равномерно сером тоне и сама выполнена в градациях серого.
2. Картинки в зоне поиска уже цветные и расположены на неравномерном случайно-градиентном сером фоне.
3. Стрелки, указывающие на варианты ответов расположены в фиксированных позициях.
4. Общий размер полотна 1600 х 400.
5. Область картинки-подсказки в левом нижнем углу размерами 130 х 200.
   
### _Шаг 2._ 
Пайплайн решения тогда будет выглядеть так:
1. Соберем и обучим первую модель YOLOv8 для распознавания предмета в зоне картинки-подсказки. Возвращать будем класс с наибольшим confidience.
2. Соберем и обучим вторую модель YOLOv8 для распознавания и классификации всех предметов на полотне. Возвращать будем классы и координаты центров bbox после детекции.
3. Найдем минимум евклидова расстояния между координатами центров 8 стрелок и координатами классов совпадающих с подсказкой - это и будет центр искомого предмет.
4. Проверим, чтобы найденная точка находилась внутри объекта true_objects из исходной json разметки - это будет успешное предсказание. Все остальные случаи - неуспешное предсказание.
5. Определим итоговую точность решения как отношение успешных предсказаний как всем попыткам. Наша цель - не менее 95%.

#### _Шаг 2.1_
Сборка датасета для обучения первой модели.
1. У нас есть вырезанные предметы. Для подсказки они нужны серыми - переведем их все в градации серого.
2. Поскольку предметы вырезались по границам рамок из json файла, то координаты аннотаций очевидно у каждого предмета теперь будут по краям его изображения.
3. Количество по классам разбалансировано, находим наибольшее число образцов в классе - 660 штук. 
4. Подготовим серую подложку 130 х 200 и поместим на ней в случайном месте предмет. Для устранения разбалансировки, количество размещений случайно выбранного из своего поднабора предмета, сделаем одинаковое число раз (по 660) в случайно выбранном месте на подложке.
5. Зная размер исходной картинки, размер подложки и величину смещения картинки на полотне, переопределяем координаты bbox в аннотациях.
6. Сделаем сплитование 80/20 по каждому классу и получаем итоговый общий датасет.

#### _Шаг 2.2_
Сборка датасета для обучения второй модели.
1. Создадим подложку 1600 х 400 и заполним ее для каждого вызова случайным перлинским шумом, имитирующим фон для зоны поиска.
2. В предложенных наборах мы видим примерно по 40-50 предметов на каждом исследуемом изображении, которые занимают верхнюю его половину. Будем заполнять учебную подложку полностью, поэтому будет 80-100 случайных предметов, раскиданных с зазорами между собой по полотну.
3. Аналогично прошлого шага фиксируем координаты bbox каждого предмета со своим классом в аннотациях.
4. Сделаем сплитование 80/20 по каждому классу и получаем итоговый общий датасет.

### _Шаг 3._
Тестирование и проверка точности.

Виды ошибок, которые могут быть допущены на этапе предсказания и засчитываются в сумму неуспешных предсказаний.
1. Неверно определен предмет в подсказке.
2. Не обнаружен никакой предмет в подсказке (no detection).
3. Неверно определен предмет в зоне поиска на месте истинного ответа.
4. Неверно определены оба предмета.
5. Неверно записаны координаты истинного ответа true_objects в json файле:
   - Координаты выходят за пределы зоны поиска - этот случай принудительно исключаю из рассмотрения и переходим к следующему входному изображению.
   - Координаты правого нижнего угла меньше координат левого верхнего угла - этот случай принудительно исключаю из рассмотрения и переходим к следующему входному изображению.
   - Расхождение графической метки с числовыми координатами, не попавшее в два предыдущих случая, выявить нельзя, поэтому принимаю такой случай как истинную ошибку.
6. При детекции второй моделью bbox определен по правильному предмету, но не соответствует его реальным границам - возможен выход предсказанного центра за пределы целевого bbox из true_objects в json файле.

## Итоги
Поскольку в процессе обучения не использовались предоставленные наборы данных в исходном виде, а были сформированы полностью свои датасеты, то для тестов допустимо использовать все исходные наборы images_100 + labels_100 (100 пар) и images_train + labels_train (3998 пар) в полном объеме.

![image](https://github.com/antonshalin76/capture_solver/assets/50358174/366677bf-f141-4b8b-9033-557f31cfe164)

__Итоговые результаты выборочных тестов: Total: 85, Correct: 66, Errors: 19 (hint not detected: 3), Accuracy: 77.65%__
