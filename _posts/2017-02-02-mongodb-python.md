---
layout: default
title: MongoDB y PyMongo
comments: true
---

Estoy realizando varios proyectos que tienen como base de datos MongoDB y estaba buscando una herramienta que me permitiese de manera rápida y eficiente realizar grandes cargas de datos para la realizar pruebas ya sea de estrés, rendimiento, conectividad, y otras pruebas funcionales.

Decidí explorar Python, lenguaje que había utilizado previamente para escribir programas para Hadoop MapReduce,  y que se utiliza bastante en el mundo del Big Data.

El primer paso es revisar que versión de MongoDB tengo,  en caso de no tener instalado MongoDB,  puedes instalarlo con tu administrador de paquetes favorito (yum, brew, etc). 

{% highlight shell %}
$ mongod --version
db version v3.2.10
git version: 79d9b3ab5ce20f51c272b4411202710a082d0317
OpenSSL version: OpenSSL 0.9.8zg 14 July 2015
allocator: system
modules: none
build environment:
   distarch: x86_64
   target_arch: x86_64
{% endhighlight %}

Puedes iniciar MongoDB escribiendo el siguiente comando:

{% highlight shell %}
$ mongod 
{% endhighlight %}


<strong>PyMongo</strong> es el driver que utilizo para conectarme desde  Python a la base de datos MongoDB.  Para revisar si lo tenemos instalados iniciamos una sesión del shell interactivo de Python:

{% highlight shell %}
$ python
Python 2.7.10 (default, Oct 23 2015, 18:05:06) 
[GCC 4.2.1 Compatible Apple LLVM 7.0.0 (clang-700.0.59.5)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
{% endhighlight %}

Una manera fácil de saber si tienes el paquete instalado es importando el client de MongoDB (MongoClient) desde PyMongo:

{% highlight python %}
>>> from pymongo import MongoClient

Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ImportError: No module named pymongo
{% endhighlight %}

Python me indica que no tengo la distribución de PyMongo instalada, así que procedí a instalarla:
{% highlight shell %}
sudo python -m easy_install pymongo
{% endhighlight %}

En caso de problemas visita <a href='http://api.mongodb.com/python/current/tutorial.html'>PyMongo Tutorail</a> para más información de como instalar el paquete PyMongo.

Estamos preparados para crear nuestro programa que genera más de dos millones de documentos para una colección que llamaremos strategies que almacena el rendimiento de diferentes estrategias dentro de una base de datos que llamaremos financial:

{% highlight python %}
# Importamos los módulos que vamos a utilizar 
from pymongo import MongoClient
from array import *
import random

# Iniciamos una conexión utilizando los valores predeterminados a nuestra base de datos local a través del puerto 27017
client = MongoClient()

# Definimos la base de datos a la que queremos conectarnos 
db = client.financial
# Borramos la colección strategies en caso de que exista
db.strategies.drop()

# Se define el arreglo strategies
strategies = []

# Llenamos el array con 100000 estrategias,  produciendo strat0 hasta strat99999 
for num in range(0,100000):
	 strategies.append('strat' + str(num))

# Inicializamos otro arreglo llamado types, que contiene los tipos de estrategias, ejempo por año, primer quarto del año, mes, etc. 36 elementos en total
types = ['Lifetime', '2012', '2013', '2014', '2015', '2016', '2012_Q1', '2012_Q2', '2012_Q3', '2012_Q4', '2012_Q[1-4]', '2012_Month[1-12]', '2013_Q1', '2013_Q2', '2013_Q3', '2013_Q4', '2013_Q[1-4]', '2013_Month[1-12]' ,'2014_Q1', '2014_Q2', '2014_Q3', '2014_Q4', '2014_Q[1-4]', '2014_Month[1-12]', '2015_Q1', '2015_Q2', '2015_Q3', '2015_Q4', '2015_Q[1-4]', '2015_Month[1-12]', '2016_Q1', '2016_Q2', '2016_Q3', '2016_Q4', '2016_Q[1-4]', '2016_Month[1-12]' ];

# Con el siguiente loop insertamos nuestros documentos utilizando insert_one y generando valores aleatorios con la función random.uniform:
for strategy in strategies:
	for tperiod in types:
		result = db.strategies.insert_one(
		{ "uuid": strategy,
		  "type": tperiod,
		  "performance": float("{0:.2f}".format(random.uniform(0.1,2.0))) 
		})

{% endhighlight %}

Abrimos una sesión de MongoDB en nuestra línea de comando y validamos cuantos registros se han creado (podemos hacerlo con Python y PyMongo también con el mismo comando):

{% highlight shell %}
$ mongo
> use financial; 
> db.strategies.count()
3960000

> db.strategies.find().limit(1)
{ "_id" : ObjectId("589154790e6fdd5565e72056"), "performance" : 1.14, "type" : "Lifetime", "uuid" : "strat0" }
{% endhighlight %}

<h3>Indices en MongoDB</h3>

La reciente carga de datos se presta para validar el funcionamiento de los índices y su eficiencia,  por defecto MongoDB cada vez que crea una colección crea un índice utilizando el campo _id como llave.

Si realizamos una búsqueda de documentos que reúnan ciertos criterios,  por ejemplo utilizando los campos type y performance,  y no definimos los índices apropiados MongoDB realizara un collection scan (COLLSCAN), que quiere decir que MongoDB examinara cada documento en la colección para seleccionar los documentos que reúnan los criterios.

Para analizar el comportamiento de una consulta en MongoDB podemos utilizar la instrucción explain, que debe ir antes de find(),  en el siguiente ejemplo utilizamos la instrucción explain(true) para analizar la consulta a todos los documentos en la colección strategies que tengan como tipo de estrategia Lifetime y con un performance mayor a 0.5

{% highlight shell %}
> db.strategies.explain(true).find({"type": "Lifetime", "performance": {$gt: 0.5}})
{% endhighlight %}

![resultado explain(true) búsqueda sin índices]({{ site.url }}/assets/pymongo_1.png)

A resaltar:
<ol>
<li>Mi consulta selecciona 86,625 documentos (puede variar en tu caso)</li>
<li>La consulta demora 2.23 segundos (¡mucho tiempo!)</li>
<li>Se examinaron todos los documentos de la colección: 3,960,000 (muy ineficiente)</li>
<li>Se utilizo COLLSCAN al no existir un índice viable a utilizar.</li>
</ol>

Ahora creamos un índice utilizando los dos campos para nuestra búsqueda:

{% highlight shell %}
> db.strategies.createIndex( {"type": 1, "performance": 1} )
{
	"createdCollectionAutomatically" : false,
	"numIndexesBefore" : 1,
	"numIndexesAfter" : 2,
	"ok" : 1
}
{% endhighlight %}

Y volvemos a ejecutar nuestra consulta:

{% highlight shell %}
> db.strategies.explain(true).find({"type": "Lifetime", "performance": {$gt: 0.5}})
{% endhighlight %}

![resultado explain(true) búsqueda con índices]({{ site.url }}/assets/pymongo_2.png) 

A resaltar:
<ol>
<li>Mi consulta selecciona 86,625 documentos (puede variar en tu caso)</li>
<li>La consulta demora 0.208 segundos</li>
<li>Se examinaron 86,625 documentos en la colección, solamente los necesarios (muy eficiente) </li>
<li>IXSCAN: Se utilizo el índice que recien creamos.</li>
</ol>
