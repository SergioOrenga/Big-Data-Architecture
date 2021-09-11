# Big-Data-Architecture-VIII
# Práctica: Sergio Orenga

# Brainstorm

-	Lucha contra la despoblación.
-	Desarrollar una web que recomiende pueblos tanto para vivir como para montar una empresa.
-	Se puede obtener información del censo de los municipios de España, así como de diferentes características interesantes, como el número de empresas que hay en cada municipio. 
-	También se puede sacar información acerca de viviendas y terrenos en venta en cada municipio.
-	Se pueden revisar redes sociales para obtener información de lo que se comenta acerca de los municipios.




# Diseño del DAaaS

## Definición la estrategia del DAaaS
La finalidad de este proyecto es el desarrollo de una aplicación web que evalúe o recomiende pueblos para vivir o para montar una empresa, considerando los filtros y preferencias del usuario, con el objetivo de evitar la despoblación. Teniendo en cuenta la media de edad de los habitantes de los municipios, el paro registrado, la cantidad y el precio de las viviendas y terrenos a la venta, etc.

## Arquitectura DAaaS

En el proyecto se van a necesitar los siguientes componentes de software:
-	Una aplicación Web (Google Cloud Platform / Compute Engine). Se trata de una máquina virtual que va a mostrar un dashboard realizado con Google Data Studio, el cual se alimenta de los datos de un servidor SQL.
      *	Una base de datos SQL (Google Cloud Platform / SQL).
      *	Un dashboard para visualización de datos (Google Data Studio).
                  
-	Un Bucket para el almacenamiento de datos en la nube (Google Cloud Platform / Cloud Storage).
-	Un cluster de Hadoop (Google Cloud Platform / Dataproc). Planificado para que se active una vez por semana.
-	Un Crawler de Web para recoger la información sobre las viviendas en venta en los municipios de España desde la página https://www.idealista.com/buscar/venta-viviendas/casa_de_pueblo_en_venta/ (Google Cloud Platform / Cloud Functions – Cloud Scheduler). Planificado para que se ejecute una vez por semana.
-	Un Crawler de Web para recoger la información sobre terrenos en venta en los municipios españoles desde la página https://terrenos.es/venta (Google Cloud Platform / Cloud Functions – Cloud Scheduler). Planificado para que se ejecute una vez por semana.
-	Descargar manualmente los siguientes archivos CSV:
      *	Padrón municipios España (https://opendata.esri.es/datasets/municipios-de-espa%C3%B1a-padron2017/explore?location=35.713866%2C-6.916495%2C5.45)
      *	Número de empresas por municipio (https://www.ine.es/jaxiT3/Tabla.htm?t=4721)
      *	Número total de transacciones inmobiliarias por municipio (https://apps.fomento.gob.es/BoletinOnline2/?nivel=2&orden=34000000)
      *	Códigos de municipios de España (https://www.ine.es/dyngs/INEbase/es/operacion.htm?c=Estadistica_C&cid=1254736177031&menu=ultiDatos&idp=1254734710990)
      *	Paro registrado por municipio (https://datos.gob.es/es/catalogo/ea0021425-paro-registrado-por-municipios)
-	Un Scrapper de Facebook con geolocalización (Google Cloud Platform / Kafka Cluster – VM Compute Engine). Se necesitan dos máquinas virtuales, una para el publicador y la otra para el suscriptor.
-	Un Scrapper de Twitter con geolocalización (Google Cloud Platform / Kafka Cluster – VM Compute Engine). Se necesitan dos máquinas virtuales, una para el publicador y la otra para el suscriptor. Son las mismas que las utilizadas para el Scrapper de Facebook.
-	Utilizar la API de “Meaning Cloud” para analizar la reputación de los municipios con lo que respecta a diferentes características establecidas (Análisis de Reputación Corporativa).

## DAaaS Operating Model Design and Rollout

La descarga de los archivos, que se efectúa manualmente, se realiza una vez al año, debido a la naturaleza de los datos, que no se actualizan frecuentemente. La descarga se realiza desde el ordenador local, y luego se suben manualmente los archivos CSV obtenidos al Bucket de Cloud Storage.

Los Crawlers, tanto de la web de terrenos en venta, como la de viviendas en venta, se ejecutan semanalmente para refrescar las propiedades ofertadas con sus características más importantes. La ejecución se realiza mediante código Python utilizando la librería Scrapy en Cloud Functions, y el activador se planifica con Cloud Scheduler. Como resultado se obtienen los archivos CSV que se almacenan en el Bucket de Cloud Storage.

Los Scrappers de Facebook y Twitter se ejecutan en todo momento, con el objetivo de capturar todos los comentarios que nos interesan independientemente del momento en el que se produzcan. Para realizar la captura de dichos datos se utiliza la herramienta de streaming Apache Kafka, la cual se compone de dos instancias de máquinas virtuales (Compute Engine), una se utiliza como “publicador” y la otra como “suscriptor”. El publicador inserta los comentarios de interés en el sistema a través de topics, y el suscriptor es el encargado de procesar los topics ingestados. El suscriptor envía los comentarios a la API de “Cloud Meaning” para realizar un análisis de reputación corporativa. Se ha adaptado el análisis de reputación corporativa para que realice el análisis de reputación de los municipios estableciendo categorías específicas de los municipios en el modelo de reputación corporativa. Las categorías definidas son variables personales, sociales, económicas, ambientales y de salud, y se extrae el sentimiento (positivo, negativo o neutro) relativo a dichas categorías. Además, esta API soporta multilenguaje, por lo que es capaz de analizar comentarios en varios idiomas. Como resultado de este análisis se obtiene un archivo CSV que se almacena en el Bucket de Cloud Storage.

La base del procesamiento de los datos del sistema es el cluster de Hadoop (Dataproc). El cluster de Hadoop lee los datos de un Bucket de Cloud Storage. En dicho Bucket se almacenan los datos de los Crawlers de viviendas y terrenos en venta, los Scrappers de Facebook y Twitter, y los archivos descargados manualmente. El cluster se arranca una vez a la semana mediante una Cloud Function que es invocada por el activador de Cloud Scheduler, que es el que planifica las activaciones. Una vez arrancado, el cluster procesa los datos y genera una estimación de la probabilidad de despoblación de cada municipio tanto a corto como a medio y largo plazo. También se crea un archivo .sql, que se almacena en el Bucket de Cloud Storage, y contiene todas las instrucciones de SQL necesarias para actualizar los datos procesados en la base de datos de Cloud SQL. Dicha actualización o sincronización de los datos de la base de datos Cloud SQL se realiza semanalmente, y es ejecutada mediante una Cloud Function, la cual es activada por un Cloud Scheduler.

La visualización de los datos de la base de datos Cloud SQL se realiza mediante un dashboard de Google Data Studio. Para poder acceder a los datos de la base de datos de Cloud SQL desde Google Data Studio es necesario utilizar el conector de cloud SQL existente en Google Data Studio. Una vez desarrollado el dashboard, se incrusta en la Web App creada en una instancia de máquina virtual (Compute Engine). Ese dashboard será con el que interactuarán los usuarios finales para realizar los filtros y agrupaciones que consideren necesarias para las consultas que necesiten.

## Desarrollo de la plataforma DAaaS.

* Los campos de las tablas que se pasan a la base de datos de Cloud SQL son los siguientes:

### Tabla municipios:
```
Codigo_municipio

Nombre_municipio

Codigo_provincia

Nombre_provincia

Numero_habitantes

Media_edad

Numero_empresas

Total_paro_registrado

Paro_hombre_edad<25

Paro_hombre_edad_25-45

Paro_hombre_edad>=45

Paro_mujer_edad<25

Paro_mujer_edad_25-45

Paro_mujer_edad>=45

Paro_agricultura

Paro_industria

Paro_construcción

Paro_servicios

Paro_sin_empleo_anterior

Reputacion_personal

Reputacion_social

Reputacion_economica

Reputacion_ambiental

Reputacion_salud

Probabilidad_despoblacion_corto_plazo

Probabilidad_despoblacion_medio_plazo

Probabilidad_despoblacion_largo_plazo
```


### Tabla propiedades_venta:
```
Codigo_municipio

Nombre_municipio

Tipo_propiedad

Titulo_propiedad

Descripcion_propiedad

Precio_propiedad

M2_propiedad

Ubicacion_propiedad

Contacto_propiedad
```


### Tabla transacciones_inmobiliarias:
```
Codigo_municipio

Nombre_municipio

Año

Trimestre

Numero_transacciones_inmobiliarias
```


* El código del Crawler de la web de terrenos en venta es el siguiente:
```Python
import scrapy
import json
from scrapy import Request

class BlogSpider(scrapy.Spider):
    name = 'blogspider'
    start_urls = ['https://terrenos.es/venta']
    
    # con esto limito no hacer más de 265 ejecuciones 
    COUNT_MAX = 265
    count = 0
    def parse(self, response):
        # Aqui scrapeamos los datos y los imprimimos a un fichero
        for article in response.css('div.card-list__item'):
            extract_precio = article.css('div.card__heading.heading.heading_type_h3.mr-2 ::text').extract_first()
            if extract_precio is not None:
              extract_precio = extract_precio.strip().replace(',', '.')
            extract_precio2 = article.css('div.card_price2.color-light.caption_size_gs ::text').extract_first()
            if extract_precio2 is not None:
              extract_precio2 = extract_precio2.strip().replace(',', '.')
            extract_tamanyo = article.css('div.card__content.grid.grid_wrap > div.col.p-0 > div:nth-child(2) ::text').extract_first()
            if extract_tamanyo is not None:
              extract_tamanyo = extract_tamanyo.strip().replace(',', '.')
            extract_ubicacion = article.css('div.card__caption.semibold.color-pumpkin.font-s.col-12.p-0 ::text').extract_first()
            if extract_ubicacion is not None:
              extract_ubicacion = extract_ubicacion.strip().replace(',', '.')
            # Print a un fichero
            print(f"{extract_precio},{extract_precio2},{extract_tamanyo},{extract_ubicacion}", file=filep)
        # Aqui hacemos crawling (con el follow)
        for next_page in response.css('div.pagination__list > a:nth-child(4)'):
            print(next_page)
            self.count = self.count + 1
            if (self.count < self.COUNT_MAX):
                yield response.follow(next_page, self.parse)


filep = open('gs://database_data_so/crawler/properties1.csv', 'w', encoding='utf-8')

from scrapy.crawler import CrawlerProcess

process = CrawlerProcess({
    'USER_AGENT': 'Google SEO Bot'
})

def main():
    process.crawl(BlogSpider)
    process.start()
    filep.close()
```

* El código del Crawler de la web de viviendas en venta en pueblos es el siguiente:
```Python
import scrapy
import json
from scrapy import Request

class BlogSpider(scrapy.Spider):
    name = 'blogspider'
    start_urls = [‘https://www.idealista.com/buscar/venta-viviendas/casa_de_pueblo_en_venta/']


    def parse(self, response):
        # Aqui scrapeamos los datos y los imprimimos a un fichero
        for article in response.css(‘article’):
            extract_titulo = article.css('div.item-info-container > a ::text').extract_first()
            if extract_titulo is not None:
              extract_titulo = extract_titulo.strip().replace(',', '.')

            extract_precio = article.css('span.item-price.h2-simulated ::text').extract_first()
            if extract_precio is not None:
              extract_precio = extract_precio.strip().replace(',', '.')

            extract_descripcion = article.css('div.item-description.description > p.ellipsis ::text').extract_first()
            if extract_descripcion is not None:
              extract_descripcion = extract_descripcion.strip().replace(',', '.')

            extract_m2 = article.css('span.item-detail ::text').extract_first()
            if extract_m2 is not None:
              extract_m2 = extract_m2.strip().replace(',', '.')

            extract_contacto = article.css('span.icon-phone.item-not-clickable-phone ::text').extract_first()
            if extract_contacto is not None:
              extract_contacto = extract_contacto.strip().replace(',', '.')

            
            # Print a un fichero
            print(f"{extract_titulo},{extract_precio},{extract_descripcion},{extract_m2},{extract_contacto}", file=filep)

        # Aqui hacemos crawling (con el follow)
        for next_page in response.css('li.next a'):
            print(next_page)
            yield response.follow(next_page, self.parse)


filep = open('gs://database_data_so/crawler/properties2.csv', 'w', encoding='utf-8')

from scrapy.crawler import CrawlerProcess

process = CrawlerProcess({
    'USER_AGENT': 'Google SEO Bot'
})

def main():
    process.crawl(BlogSpider)
    process.start()
    filep.close()
```

# Link a Diagrama:

https://docs.google.com/drawings/d/1-7sfOqKQzjpmjLPuElQOn6_FdbSkBzBjsikUNwxnXF0/edit?usp=sharing

