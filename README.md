# GIS технологии

## 1. PostGIS
**[PostGIS Tutorial](https://postgis.net/workshops/postgis-intro/)**
#### PostGIS — это расширение с открытым исходным кодом для PostgreSQL. Он привносит в PostgreSQL три вещи, которые имеют решающее значение для работы с пространственными данными:
- Поддержка типов пространственных данных (точки, линии, полигоны, растры).
- Пространственные функции (подробнее о них ниже).
- Пространственное индексирование, чтобы ваши пространственные запросы выполнялись быстро.

PostGIS умеет работать с двумя типами координат: **GEOMETRY** и **GEOGRAPHY**. Вот в чем их разница:
- **GEOMETRY** — для «плоских» вычислений, где форма Земли не учитывается. Это оптимально для локальных данных (например, в пределах одного города).
- **GEOGRAPHY** — для сферических вычислений. Этот тип позволяет учитывать кривизну Земли и использовать геодезические координаты, что хорошо впишется для глобальных задач (например, расчёта расстояния между городами на разных континентах).
### Типы геометрий
#### Point
A spatial point represents a single location on the Earth. This point is represented by a single coordinate (including either 2-, 3- or 4-dimensions). Points are used to represent objects when the exact details, such as shape and size, are not important at the target scale. For example, cities on a map of the world can be described as points, while a map of a single state might represent cities as polygons.
```sql
SELECT ST_AsText(geom)
  FROM geometries
  WHERE name = 'Point';
```
```sql
POINT(0 0)
```
Some of the specific spatial functions for working with points are:
- ST_X(geometry) returns the X ordinate
- ST_Y(geometry) returns the Y ordinate
#### LineString
A linestring is a path between locations. It takes the form of an ordered series of two or more points. Roads and rivers are typically represented as linestrings. A linestring is said to be closed if it starts and ends on the same point. It is said to be simple if it does not cross or touch itself (except at its endpoints if it is closed). A linestring can be both closed and simple.

The following SQL query will return the geometry associated with one linestring (in the ST_AsText column).
```sql
SELECT ST_AsText(geom)
  FROM geometries
  WHERE name = 'Linestring';
```
```sql
LINESTRING(0 0, 1 1, 2 1, 2 2)
```
Some of the specific spatial functions for working with linestrings are:
- ST_Length(geometry) returns the length of the linestring
- ST_StartPoint(geometry) returns the first coordinate as a point
- ST_EndPoint(geometry) returns the last coordinate as a point
- ST_NPoints(geometry) returns the number of coordinates in the linestring
#### Polygon
```sql
SELECT ST_AsText(geom)
  FROM geometries
  WHERE name LIKE 'Polygon%';
```
```sql
POLYGON((0 0, 1 0, 1 1, 0 1, 0 0))
POLYGON((0 0, 10 0, 10 10, 0 10, 0 0),(1 1, 1 2, 2 2, 2 1, 1 1))
```
Some of the specific spatial functions for working with polygons are:
- ST_Area(geometry) returns the area of the polygons
- ST_NRings(geometry) returns the number of rings (usually 1, more if there are holes)
- ST_ExteriorRing(geometry) returns the outer ring as a linestring
- ST_InteriorRingN(geometry,n) returns a specified interior ring as a linestring
- ST_Perimeter(geometry) returns the length of all the rings
#### Collection
There are four collection types, which group multiple simple geometries into sets.
- **MultiPoint**, a collection of points
- **MultiLineString**, a collection of linestrings
- **MultiPolygon**, a collection of polygons
- **GeometryCollection**, a heterogeneous collection of any geometry (including other collections)
Our example collection contains a polygon and a point:
```sql
SELECT name, ST_AsText(geom)
  FROM geometries
  WHERE name = 'Collection';
```
```sql
GEOMETRYCOLLECTION(POINT(2 0),POLYGON((0 0, 1 0, 1 1, 0 1, 0 0)))
```
### Создание геометрии
Чтобы начать работу, создаем таблицу и добавляем GEOGRAPHY или GEOMETRY поля. Пример на GEOGRAPHY:
```sql
CREATE TABLE cafes (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    location GEOGRAPHY(Point, 4326)
);
```
Здесь GEOGRAPHY(Point, 4326) задаёт тип Point в системе координат 4326 (WGS 84, которая чаще всего используется в GPS и на картах).
### Вставка данных: ST_SetSRID и ST_MakePoint
Функции ST_MakePoint и ST_SetSRID — это базовые инструменты для работы с точками в PostGIS:
- **ST_MakePoint(x, y)** — создаёт точку с координатами x (долгота) и y (широта). Но эта точка будет без системы координат.
- **ST_SetSRID(geometry, srid)** — устанавливает систему координат для объекта. Для глобальных координат мы используем SRID 4326 (WGS 84).

Пример вставки точек:
```sql
INSERT INTO cafes (name, location)
VALUES
('Кафе А', ST_SetSRID(ST_MakePoint(37.6173, 55.7558), 4326)),
('Кафе Б', ST_SetSRID(ST_MakePoint(30.3351, 59.9343), 4326));
```
### Индексы: GIST и SP‑GiST
Чтобы ускорить пространственные запросы, создаём индекс. 

В PostGIS для геоданных используются индексы типа GIST и SP-GiST, которые оптимизированы для поиска по координатам и обработке пространственных данных.
```sql
CREATE INDEX idx_cafes_location ON cafes USING GIST (location);
```
Без индекса запросы на больших данных будут работать медленно. GIST — более распространённый индекс для большинства пространственных запросов, SP-GiST используется реже, но может в целом хорош для специфических структур данных, например, для структур данных, которые имеют иерархическую природу или нерегулярные разделения, например, для работы с неравномерными географическими зонами или деревьями.
### Поиск ближайших объектов: ST_Distance и ST_DWithin
Допустим, задача — найти ближайшие кафе в радиусе 5 км от заданной точки. Здесь хороши две функции:
- **ST_Distance(geometry, geometry)** — вычисляет расстояние между двумя объектами. Если тип данных GEOGRAPHY, результат будет в метрах.
- **ST_DWithin(geometry, geometry, distance)** — возвращает true, если объекты находятся в пределах заданного расстояния. Отличается от ST_Distance тем, что позволяет сразу фильтровать результаты.

Пример запроса на ближайшие кафе в радиусе 5 км:
```sql
SELECT name, ST_Distance(location, ST_SetSRID(ST_MakePoint(37.6173, 55.7558), 4326)) AS distance
FROM cafes
WHERE ST_DWithin(location, ST_SetSRID(ST_MakePoint(37.6173, 55.7558), 4326), 5000)
ORDER BY distance;
```
Здесь:
- Используем ST_DWithin, чтобы найти все кафе в радиусе 5 км от точки (37.6173, 55.7558).
- Сортируем результат по расстоянию (для этого используем ST_Distance).
### Построение буферов и проверка пересечений
Помимо поиска ближайших объектов, PostGIS поддерживает буферные зоны и операции с полигонами. Например:
- **ST_Buffer(geometry, distance)** — создаёт буферную зону вокруг объекта (например, круг радиусом distance метров вокруг точки).
- **ST_Intersects(geometry, geometry)** — проверяет, пересекаются ли два объекта.

Пример: проверка, попадает ли кафе в буфер 2 км от точки:
```sql
SELECT name
FROM cafes
WHERE ST_Intersects(location, ST_Buffer(ST_SetSRID(ST_MakePoint(37.6173, 55.7558), 4326), 2000));
```
Этот запрос покажет кафе, которые находятся в пределах 2 км от заданной точки, используя буферную зону.
### ST_Transform
- **ST_SetSRID** не меняет координаты, но добавляет метаданные, чтобы указать, в какой пространственной системе отсчета на самом деле находятся координаты.
- **ST_Transform** используется для изменения базовых координат из известной системы пространственной привязки в другую известную систему пространственной привязки.
#### Примеры
You forgot to specify the spatial reference system of your data or specified it wrong, but you know its WGS 84 long lat:
```sql
ALTER TABLE mytable
 ALTER COLUMN geom TYPE geometry(MultiPolygon, 4326)
  USING ST_SetSRID(geom, 4326);
```
Your data is WGS 84 long lat, and you tagged it correctly but you want it in US National Atlas meters:
```sql
ALTER TABLE mytable
  ALTER COLUMN geom
   TYPE geometry(MultiPolygon, 2163)
  USING ST_Transform(geom, 2163);
```
### Пример создания таблицы (Django)
В Django с расширением postgis можно работать, если поменять engine в сетапе для постгреса на **'django.contrib.gis.db.backends.postgis'**
#### Хранение геометрии отрезка
```python
from django.contrib.gis.db import models

class Data(models.Model):
    geom = models.LineStringField(srid=3857, null=False, verbose_name='геометрия отрезка')
    avg_cnt = models.IntegerField(null=False, verbose_name='среднее количество пешеходов на отрезке')
```
#### Хранение геометрии точки
```python
from django.contrib.gis.db import models

class Data(models.Model):
    geom = models.PointField(srid=3857, null=False, verbose_name='точка')
    name = models.CharField(max_length=200, null=False, blank=False, verbose_name='название объекта')
```

## 2. Форматы wkt, ewkt, wkb, ewkb
### WKT
**WKT** — это стандартный текстовый формат для описания геометрических объектов, определенный в спецификации OGC (Open Geospatial Consortium). Этот формат позволяет легко хранить, передавать и интерпретировать геометрические данные.
Примеры WKT:

Точка (POINT):
```python
POINT(30 10)
```
Это точка с координатами (30, 10).

Линия (LINESTRING):
```python
LINESTRING(30 10, 10 30, 40 40)
```
Это линия, проходящая через три точки: (30, 10), (10, 30) и (40, 40).

Полигон (POLYGON):
```python
POLYGON((30 10, 40 40, 20 40, 10 20, 30 10))
```
Это замкнутый полигон с пятью вершинами. Первая и последняя точки совпадают, чтобы закрыть форму.

Многоугольник (MULTIPOLYGON):
```python
MULTIPOLYGON(((30 20, 45 40, 10 40, 30 20)), ((15 5, 40 10, 10 20, 5 10, 15 5)))
```
Это коллекция из двух полигонов.
### EWKT
**EWKT** — это расширение формата WKT, которое добавляет дополнительную информацию о пространственной ссылке (Spatial Reference System, SRS) и/или метаданные к геометрическому объекту. EWKT часто используется в системах, таких как PostGIS, для явного указания системы координат.

Точка с SRID:
```python
SRID=4326;POINT(30 10)
```
Это точка с координатами (30, 10) в системе координат WGS84 (EPSG:4326).

Преимущества использования WKT и EWKT
- Простота: Форматы WKT и EWKT легки для чтения и понимания человеком.
- Стандартизация: WKT является частью стандарта OGC, что гарантирует совместимость между различными ГИС-системами.
- Гибкость: EWKT позволяет явно указывать систему координат, что уменьшает вероятность ошибок при работе с геоданными.
- Универсальность: Эти форматы поддерживаются большинством современных геоинформационных систем и баз данных.
### WKB
Форматы **WKB (Well-Known Binary)** и **EWKB (Extended Well-Known Binary)** являются бинарными представлениями геометрических объектов, которые используются для хранения и передачи пространственных данных. Они являются аналогами текстовых форматов WKT и EWKT, но более компактны и эффективны в обработке.

#### WKB — это стандартный бинарный формат для описания геометрических объектов, определенный в спецификации OGC (Open Geospatial Consortium). Этот формат используется для хранения геометрических данных в базах данных, таких как PostgreSQL с расширением PostGIS, а также для передачи данных между системами.
Для точки (30, 10) в Little Endian формате WKB будет выглядеть так:
```python
010100000000000000000024400000000000002440
```
- 01: Little Endian.
- 01: Тип геометрии (POINT).
- 0000000000002440: Координата X (30).
- 0000000000002440: Координата Y (10).
### EWKB
#### EWKB — это расширение формата WKB, которое добавляет информацию о системе координат (Spatial Reference System, SRS) или другие метаданные к геометрическому объекту. EWKB часто используется в системах, таких как PostGIS, для явного указания системы координат.
Пример EWKB для точки (1000000, 5000000) в EPSG:3857:
```python
01010000A0110F000000000000E8037B410000000000008AFF
```
Разбор EWKB:
- 01: Little Endian.
- 01: Тип геометрии (POINT).
- A0110F0000: SRID (3857 в Little Endian).
- 00000000E8037B41: Координата X (1000000 в формате IEEE 754 double precision).
- 0000000000008AFF: Координата Y (5000000 в формате IEEE 754 double precision).

## 3. Проекции 3857 4326
**EPSG:4326** — географическая система координат, основанная на системе параметров WGS84. Единица измерения — градус.

**EPSG:3857** — прямоугольная система координат, основанная на проекции Меркатора, построенной по системе параметров WGS84. Единица измерения – метр.
#### EPSG:3857 — это плоская проекция, известная как Web Mercator или Spherical Mercator . Она преобразует сферическую поверхность Земли в плоскость, что позволяет использовать ее для веб-карт.
#### Хорошо подходит для веб-карт, так как обеспечивает равномерное масштабирование при увеличении/уменьшении.

## 4. Векторные и растровые форматы (карты)
- **Векторные данные** – это тип географических данных, в котором информация хранится в виде набора точек, линий или полигонов, а также атрибутивных данных этих объектов.
- **Растровые данные** – тип географических данных, в котором информация хранится в виде сетки из пикселей регулярного размера, и атрибутивные данные присвоены каждому пикселю. (Карта в виде спутникового снимка)
![](1.png?raw=true "Title")

## 5. Библиотека h3
#### H3 — это шестиугольная система геопространственного индексирования, разработанная компанией Uber. Она позволяет разбивать земную поверхность на шестиугольники различных размеров и агрегировать в них данные, что помогает эффективно визуализировать геоданные и проводить их быстрый анализ.
- Преобразование координат в индексы:

Функция h3.geo_to_h3(lat, lng, resolution) преобразует широту (lat) и долготу (lng) в индекс H3 заданного уровня детализации.

- Преобразование индексов обратно в координаты:

Функция h3.h3_to_geo(h3_index) возвращает центральную точку ячейки в виде широты и долготы.

- Поиск соседних ячеек:

Функция h3.k_ring(h3_index, k) находит все ячейки, находящиеся в радиусе k шагов от заданной ячейки.

- Проверка принадлежности:

Функция h3.h3_is_valid(h3_index) проверяет, является ли индекс H3 допустимым.

- Расчет расстояний:

Функция h3.h3_distance(h3_index1, h3_index2) вычисляет минимальное количество шагов между двумя ячейками.

## 6. Тайлы (Tiles)
Обычно карта разрезается на тайлы – участки размером 256x256 пикселей. Каждый тайл хранится в отдельном PNG-файле. Это позволяет приложениям экономить трафик при загрузке карты: можно подгружать не всю карту сразу, а только те тайлы, которые нужно показать в данный момент.

Каждый тайл имеет порядковый номер [x, y], где x – номер тайла по оси X, y – по оси Y. Нумерация тайлов начинается от левого верхнего угла карты мира и продолжается вправо вниз. Отсчет начинается с нуля.
![](2.png?raw=true "Title")

Тайлы создаются отдельно для каждого уровня масштабирования карты:

- На наименьшем уровне (zoom=0) вся территория покрывается одним тайлом (см. рисунок ниже). 
- На следующем уровне (zoom=1) вся карта умещается на четырех тайлах. 
- При zoom=2 карту покрывает сетка из 16 тайлов, и так далее. 
- Таким образом, на каждом шаге масштабирования карта разбита на 4z тайлов, где z — уровень масштабирования.
![](3.png?raw=true "Title")

## 7. protobuf / vnd.mapbox-vector-tile
#### Формат application/vnd.mapbox-vector-tile — это спецификация для векторных тайлов, разработанная компанией Mapbox . 
Этот формат используется для передачи географических данных в виде векторных тайлов, которые затем могут быть отображены на картах с помощью различных клиентских библиотек и фреймворков. 

Векторные тайлы обеспечивают более эффективную передачу и обработку данных по сравнению с растровыми тайлами.

Этот формат основан на стандарте Mapbox Vector Tile Specification , который использует бинарный формат **Protocol Buffers (Protobuf)** для представления данных. Благодаря компактности и эффективности Protobuf, векторные тайлы занимают меньше места и быстрее передаются через сеть.

## 8. shapefile .shp
#### Shapefile — это один из самых распространенных форматов для хранения и обмена векторных географических данных. Разработанный компанией Esri (Environmental Systems Research Institute), этот формат используется в геоинформационных системах (ГИС) для представления точек, линий и полигонов, а также связанных с ними атрибутов.
Несмотря на название, Shapefile на самом деле представляет собой набор нескольких файлов , каждый из которых выполняет определенную функцию. Эти файлы обычно хранятся вместе в одной директории.

Основные файлы Shapefile:
- .shp (Shape Format):
Содержит геометрические объекты (точки, линии, полигоны).
Это основной файл, который определяет пространственную информацию.
- .shx (Shape Index Format):
Содержит индекс геометрических объектов из файла .shp.
Позволяет быстро находить нужные объекты.
- .dbf (Attribute Format):
Хранит атрибутные данные в формате dBASE.
Каждая строка соответствует одному геометрическому объекту из файла .shp.

Дополнительные файлы (необязательные):
- .prj (Projection Format):
Описывает систему координат (например, EPSG:4326 или EPSG:3857).
Без этого файла система координат неизвестна.
- .sbn и .sbx (Spatial Index):
Содержат пространственный индекс для ускорения поиска объектов.
- .xml (Metadata):
Хранит метаданные о датасете.
- .cpg (Code Page):
Указывает кодировку текстовых данных (например, UTF-8).

#### Работа с shapefile в python
```python
import geopandas as gpd

# Чтение Shapefile
gdf = gpd.read_file("path/to/file.shp")

# Просмотр данных
print(gdf.head())

# Сохранение в новый файл
gdf.to_file("path/to/output.shp")
```
## 9. GeoPackage .gpkg
#### GeoPackage — это единый файл в формате SQLite, который содержит все необходимые данные для работы с географической информацией. В отличие от Shapefile, где данные распределены между несколькими файлами, GeoPackage объединяет всё в одном файле, что упрощает управление данными.

Основные характеристики GeoPackage:
- Единый файл:
Все данные (геометрия, атрибуты, метаданные, растры) хранятся в одном файле .gpkg.
- Поддержка различных типов данных:
Векторные данные (точки, линии, полигоны).
Растровые данные.
Метаданные.
Пространственные индексы.
- Расширяемость:
GeoPackage поддерживает пользовательские таблицы и расширения, что делает его гибким для различных задач.
- Поддержка временных данных:
Можно хранить информацию со временными метками (например, траектории объектов).
- Кроссплатформенность:
GeoPackage работает на всех платформах, поддерживающих SQLite.
- Стандарт OGC:
GeoPackage является официальным стандартом OGC, что гарантирует совместимость между различными ГИС-системами.

#### Структура GeoPackage
GeoPackage основан на базе данных SQLite и имеет определенную структуру:
- Таблицы для векторных данных:
Каждый слой хранится в отдельной таблице.
Таблицы содержат столбцы для геометрии (geometry) и атрибутов.
- Метаданные:
Специальные таблицы для описания содержимого файла (например, gpkg_contents).
- Пространственный индекс:
Автоматически создаваемый индекс для ускорения поиска геометрических объектов.
- Растры:
Поддержка хранения растровых данных в специальных таблицах.

#### Работа с geopackage в python
```python
import geopandas as gpd

# Чтение GeoPackage
gdf = gpd.read_file("path/to/file.gpkg", layer="points")

# Просмотр данных
print(gdf.head())

# Сохранение в новый файл
gdf.to_file("path/to/output.gpkg", driver="GPKG")
```

## 10. GeoJSON
#### GeoJSON — это открытый стандартный формат, предназначенный для представления географических данных в структурированном виде с использованием JSON (JavaScript Object Notation). 
Он широко используется для хранения и передачи геопространственной информации, такой как точки, линии, полигоны и другие геометрические объекты, а также их атрибутивных данных.

Пример GeoJSON:
```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "geometry": {
        "type": "Point",
        "coordinates": [102.0, 0.5]
      },
      "properties": {
        "name": "Пример точки"
      }
    },
    {
      "type": "Feature",
      "geometry": {
        "type": "Polygon",
        "coordinates": [
          [
            [100.0, 0.0],
            [101.0, 0.0],
            [101.0, 1.0],
            [100.0, 1.0],
            [100.0, 0.0]
          ]
        ]
      },
      "properties": {
        "name": "Пример полигона"
      }
    }
  ]
}
```
Применение:
- Веб-картография: GeoJSON часто используется в веб-приложениях для отображения данных на картах (например, с помощью библиотек Leaflet или Mapbox).
- ГИС-системы: многие геоинформационные системы поддерживают импорт и экспорт данных в формате GeoJSON.
- API: многие сервисы, такие как OpenStreetMap, Google Maps и другие, используют GeoJSON для передачи геоданных.

#### Для больших наборов данных GeoJSON может быть неэффективен из-за увеличения размера файла.

## 11. GEOTIFF
**GeoTIFF** — это расширение формата TIFF (Tagged Image File Format), предназначенное для хранения геопространственных данных, таких как растровые изображения, связанные с географической привязкой. Этот формат позволяет включать в файл метаданные, которые описывают местоположение, проекцию, масштаб и другие параметры, необходимые для корректного отображения изображения на карте.

## 12. KML
**KML** представляет собой текстовый файл, написанный в формате XML, который описывает географические объекты, такие как точки, линии, полигоны, модели 3D и другие пространственные данные. Файлы KML обычно имеют расширение .kml, а их сжатая версия называется KMZ (с расширением .kmz).

Пример точки:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<kml xmlns="http://www.opengis.net/kml/2.2">
  <Document>
    <Placemark>
      <name>Москва</name>
      <description>Столица России</description>
      <Point>
        <coordinates>37.6184,55.7558</coordinates>
      </Point>
    </Placemark>
  </Document>
</kml>
```
Работа с KML в Python:
```python
import simplekml

# Создание нового KML-файла
kml = simplekml.Kml()

# Добавление точки
point = kml.newpoint(name="Москва", coords=[(37.6184, 55.7558)])
point.description = "Столица России"

# Сохранение файла
kml.save("example.kml")
```

## 13. OSM (Open Street Map)
### Open Street Map, или OSM — это как Википедия, только в области картографии. Вносить правки и обновления в существующую базу, дополняя карты всего мира, может каждый зарегистрированный пользователь.

OpenStreetMap — это база данных, содержащая информацию о дорогах, зданиях, природных объектах, административных границах и других географических элементах. Все данные создаются и поддерживаются волонтерами, использующими спутниковые снимки, GPS-устройства, фотографии и другие источники информации.

## 14. shapely

## 15. gdal

## QGIS

## mapbox

## geopandas

## gdal

## graphhopper

## geoserver (ImagePyramid)

## plotly и matplotlib

