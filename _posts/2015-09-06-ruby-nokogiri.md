---
layout: default
title: Procesando documentos HTML con Nokogiri
comments: true
---

<a href="http://www.nokogiri.org/">Nokogiri</a> es una librería que nos permite leer documentos HTML/XML y procesarlos según nuestras necesidades utilizando selectores CSS y XPath.

Para demostrar su funcionamiento, que a propósito es muy sencillo,  vamos a crear un proyecto en donde leeremos de un documento HTML en línea y buscaremos en su contenido información relevante y cargaremos dicha información a una base de datos en PostgreSQL.

El documento está en <a href="https://money.cnn.com/data/dow30/">money.cnn.com</a> y despliega en tiempo real la información relacionada a las 30 compañías que componen el promedio industrial Dow Jones.

Para almacenar nuestros datos crearemos una base de datos llamada _dowjonesindustrialaverage_,  una sola tabla que almacenara las cotizaciones y sus promedios (_dowjonesindustrial_) y una función que se encargara de hacer las actualizaciones a la tabla:

{% gist 0bad7aeeb1dea446eadc %}

Asumiendo que ya tenemos instalado el gem pg (la interface que permite conectarnos a PostgresSQL), instalamos el gem de Nokogiri:

{% highlight ruby %}
gem install nokogiri
{% endhighlight %}

Creamos un archivo llamado dowJones.rb y agregamos las siguientes 4 líneas al principio:

{% highlight ruby %}
require 'rubygems'
require 'nokogiri'
require 'open-uri'
require 'pg'
{% endhighlight %}

Examinamos el código fuente del documento e identificamos que la data que buscamos esta contenida dentro una tabla que podemos seleccionar utilizando el selector wsod_dataTable y que con el selector tr podemos seleccionar cada una de las filas dentro de la tabla. 

![Selectores CSS identificados]({{ site.url }}/assets/nokogiri_fig_1.png)

Cargamos todas las filas de la tabla seleccionada previamente a un objeto de tipo Nokogori (una colección de objetos) que llamamos _tableTr_ y empezamos a recorrerlo uno por uno.

{% highlight ruby %}
page = Nokogiri::HTML(open('http://money.cnn.com/data/dow30/'))
table = page.css('.wsod_dataTable')
tableTr = table.css('tr')
{% endhighlight %}

Descartamos la primera fila,  pues contiene solo los encabezados.  Las siguientes filas contienen la información que hemos estado buscando,  recorremos el arreglo uno por uno,  identificamos que el primer selector CSS wsod_firstCol contiene el símbolo y nombre de la compañía.

{% highlight ruby %}
tableTr.each do |row|
      symbol = row.css('.wsod_firstCol').css('a').text
      coName = row.css('.wsod_firstCol').css('span').text
      lastPrice = row.css('.wsod_aRight')[0].text
      change = row.css('.wsod_aRight')[1].text
      changePct = row.css('.wsod_aRight')[2].text
      volume = row.css('.wsod_aRight')[3].text
      changeYTD = row.css('.wsod_aRight')[4].text
      
      if !symbol.to_s.empty?
        c.insertOrUpdateCompany(symbol, coName, lastPrice.delete(',').to_f, change.delete(',').to_f,
                                changePct.delete(',').to_f, volume.delete(',').to_i, changeYTD.delete(',').to_f)
     end

end
{% endhighlight %}

El siguiente selector CSS  wsod_aRight se repite  en cada interacción 5 veces y también contiene información relevante,  el precio,  el cambio,  porcentaje del cambio, etc. Obtenemos valores directamente de cada selector a través de su índice y se asignan  a su respectiva variable. 

Utilizando la función pl/psql _insertOrUpdateCompany_ , que creamos al principio, insertamos o actualizamos registros en nuestra base de datos.


Ejecutamos nuestro script: 

{% highlight ruby %}
ruby dowJones.rb
{% endhighlight %}

Consultamos nuestra tabla _dowjonesindustrial_ y nos aseguramos que nuestros datos estén cargados:

![resultados dowjonesindustrial]({{ site.url }}/assets/nokogiri_fig_2.png)

El script completo:

{% gist 0bad7aeeb1dea446eadc %}

