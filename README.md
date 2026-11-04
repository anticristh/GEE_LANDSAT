# GEE_LANDSAT
Google Earth Engine

/***************************************************************
 ANALISIS MULTITEMPORAL DE IMAGENES SATELITALES
 SECTOR: NAMIREZ BAJO - CUMBARATZA
 CANTON: ZAMORA
 PROVINCIA: ZAMORA CHINCHIPE - ECUADOR

 PERIODO: 1984 - 2026
 SENSORES: LANDSAT 5, 7, 8 Y 9
 RESOLUCION ESPACIAL: 30 METROS
 COMPOSICION: COLOR NATURAL RGB
 METODO: COMPOSICION ANUAL POR MEDIANA
 FORMATO: GEOTIFF
***************************************************************/


// ============================================================
// 1. UBICACION Y AREA DE ESTUDIO
// ============================================================

var punto = ee.Geometry.Point([-78.8493, -3.95917]);

// Area aproximada de estudio: 2 km de radio.

var area = punto.buffer(2000);

Map.centerObject(area, 15);

Map.addLayer(
  area,
  {color: 'yellow'},
  'Area de estudio'
);

Map.addLayer(
  punto,
  {color: 'red'},
  'Centro Namirez Bajo'
);


// ============================================================
// 2. FUNCION DE PREPARACION LANDSAT 5 Y 7
// ============================================================

function preparar57(img) {

  var qa = img.select('QA_PIXEL');

  // Mascara de pixeles de relleno, nubes,
  // sombras y nieve.

  var mask = qa.bitwiseAnd(1 << 0).eq(0)
    .and(qa.bitwiseAnd(1 << 1).eq(0))
    .and(qa.bitwiseAnd(1 << 3).eq(0))
    .and(qa.bitwiseAnd(1 << 4).eq(0))
    .and(qa.bitwiseAnd(1 << 5).eq(0))
    .and(img.select('QA_RADSAT').eq(0));

  var imagen = img
    .select(
      ['SR_B3', 'SR_B2', 'SR_B1'],
      ['R', 'G', 'B']
    )
    .multiply(0.0000275)
    .add(-0.2)
    .updateMask(mask);

  return imagen.copyProperties(
    img,
    ['system:time_start']
  );
}


// ============================================================
// 3. FUNCION DE PREPARACION LANDSAT 8 Y 9
// ============================================================

function preparar89(img) {

  var qa = img.select('QA_PIXEL');

  // Mascara de pixeles de relleno, nubes dilatadas,
  // cirros, nubes, sombras y nieve.

  var mask = qa.bitwiseAnd(1 << 0).eq(0)
    .and(qa.bitwiseAnd(1 << 1).eq(0))
    .and(qa.bitwiseAnd(1 << 2).eq(0))
    .and(qa.bitwiseAnd(1 << 3).eq(0))
    .and(qa.bitwiseAnd(1 << 4).eq(0))
    .and(qa.bitwiseAnd(1 << 5).eq(0))
    .and(img.select('QA_RADSAT').eq(0));

  var imagen = img
    .select(
      ['SR_B4', 'SR_B3', 'SR_B2'],
      ['R', 'G', 'B']
    )
    .multiply(0.0000275)
    .add(-0.2)
    .updateMask(mask);

  return imagen.copyProperties(
    img,
    ['system:time_start']
  );
}


// ============================================================
// 4. COLECCIONES LANDSAT COLLECTION 2 LEVEL 2
// ============================================================

var L5 = ee.ImageCollection(
  'LANDSAT/LT05/C02/T1_L2'
)
.filterBounds(area)
.filterDate('1984-03-16', '2012-05-06');


var L7 = ee.ImageCollection(
  'LANDSAT/LE07/C02/T1_L2'
)
.filterBounds(area)
.filterDate('1999-05-28', '2026-11-02');


var L8 = ee.ImageCollection(
  'LANDSAT/LC08/C02/T1_L2'
)
.filterBounds(area)
.filterDate('2013-03-18', '2026-11-02');


var L9 = ee.ImageCollection(
  'LANDSAT/LC09/C02/T1_L2'
)
.filterBounds(area)
.filterDate('2021-10-31', '2026-11-02');


// ============================================================
// 5. APLICAR PREPROCESAMIENTO
// ============================================================

var coleccion57 = L5.merge(L7)
  .map(preparar57);

var coleccion89 = L8.merge(L9)
  .map(preparar89);


// ============================================================
// 6. UNIFICAR COLECCIONES
// ============================================================

var landsat = coleccion57.merge(coleccion89);

print(
  'TOTAL DE IMAGENES LANDSAT DISPONIBLES:',
  landsat.size()
);

print('LANDSAT 5:', L5.size());
print('LANDSAT 7:', L7.size());
print('LANDSAT 8:', L8.size());
print('LANDSAT 9:', L9.size());


// ============================================================
// 7. PARAMETROS DE VISUALIZACION RGB
// ============================================================

var visual = {
  bands: ['R', 'G', 'B'],
  min: 0.015,
  max: 0.25,
  gamma: 1.15
};


// ============================================================
// 8. FUNCION DE PROCESAMIENTO ANUAL
// ============================================================

function procesarAnio(anio) {

  var inicio = ee.Date.fromYMD(anio, 1, 1);

  var fin = ee.Date.fromYMD(anio + 1, 1, 1);

  // Primer año de disponibilidad.

  if (anio === 1984) {
    inicio = ee.Date('1984-03-16');
  }

  // Último año de consulta.

  if (anio === 2026) {
    fin = ee.Date('2026-11-02');
  }

  // Seleccionar imagenes del año.

  var coleccionAnual = landsat.filterDate(
    inicio,
    fin
  );

  // Verificar disponibilidad de imagenes.

  coleccionAnual.size().evaluate(function(cantidad) {

    print(
      'AÑO ' + anio + ' - IMAGENES DISPONIBLES:',
      cantidad
    );

    if (cantidad === 0) {

      print(
        'NO EXISTEN IMAGENES DISPONIBLES PARA EL AÑO ' + anio
      );

      return;
    }

    // Composicion anual por mediana.

    var imagen = coleccionAnual
      .median()
      .clip(area);

    // Visualizacion en Google Earth Engine.

    Map.addLayer(
      imagen,
      visual,
      'NAMIREZ BAJO LANDSAT ' + anio,
      false
    );

    // Exportacion anual a 30 metros.

    Export.image.toDrive({

      image: imagen,

      description:
        'NAMIREZ_LANDSAT_' + anio,

      folder:
        'NAMIREZ_BAJO_LANDSAT_1984_2026',

      fileNamePrefix:
        'NAMIREZ_BAJO_LANDSAT_' + anio,

      region: area,

      scale: 30,

      crs: 'EPSG:32717',

      maxPixels: 1e13,

      fileFormat: 'GeoTIFF'

    });

    print(
      'EXPORTACION PREPARADA PARA EL AÑO:',
      anio
    );

  });

}


// ============================================================
// 9. PROCESAMIENTO DESDE 1984 HASTA 2026
// ============================================================

for (var anio = 1984; anio <= 2026; anio++) {

  procesarAnio(anio);

}


// ============================================================
// 10. INFORMACION DEL ESTUDIO
// ============================================================

print('==========================================');

print('ESTUDIO MULTITEMPORAL NAMIREZ BAJO');

print('Periodo: 1984 - 2026');

print('Sensores: Landsat 5, 7, 8 y 9');

print('Resolucion espacial: 30 metros');

print('Composicion: Color natural RGB');

print('Metodo: Composicion anual por mediana');

print('Sistema de coordenadas: EPSG:32717');

print('Formato de exportacion: GeoTIFF');

print('Fecha final de consulta: 01-11-2026');

print('==========================================');
