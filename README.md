

# High-Resolution Soil Moisture Retrieval Using C-Band Radar Sentinel-1 and Cosmic-ray Neutron Sensor

The primary objective of this study is to highlight a method for estimating soil moisture on a large scale by combining satellite data and field data from a CRNS. The second objective was to show the potential of using satellite data for a holistic analysis of agricultural systems by integrating into a platform, agricultural and climatic data (rainfall, evapotranspiration, crop calendar) and soil moisture in an automated way available at any scale as quasi-real-time web-application. This approach is possible due to advances in cloud computing technology and the availability of satellite data processing services on a global scale, such as Google Earth Engine, Amazon Web Service (AWS), Sentinel Hub etc. This study will describe the pipeline for retrieving soil moisture from C-band Sentinel-1 SAR data and calibration of the conversion model with CRNS.  This method is based on the modified version of the change detection-based approaches to soil moisture estimation techniques using SAR (Bauer-Marschallinger et al., 2019, 2021; Hornacek et al., 2012) .

## Bolivia Example 

#### Steps 

1. Extract 1 year lulc class probs from dynamics world dataset which is  a temporal  LULC  based on deep learning 
2. Mask  the building, open surface water and forest area 
3. Define 1 year  time series using  date class 
4. load the region of interest  and clip the satellite data
5. Define intercept and slope which were derived using the CRNS data
6. Apply  sentinel 1 radar data processing and boxcar for smoothing and reducing  noise
7. run clustering analysis for better visualization between wet and dry region 

## Soil Moisture Physics  

![Soil Physics Principal](https://github.com/mmbaye/Bolivia-example/blob/main/figures/SoilMoisurePhysics.jpeg)


```javascript
var region=ee.FeatureCollection('users/chaponda/Tuni-Condoriri-HuaynaPotosi-Basin')
var dwVisParams = {
  min: 0,
  max: 8,
  palette: [
    '#419BDF', '#397D49', '#88B053', '#7A87C6', '#E49635', '#DFC35A',
    '#C4281B', '#A59B8F', '#B39FE1'
  ]
};

var now=ee.Date(Date.now())

var intercept=0.5
var slope=0.0222

/******************************************************************************
 * and extract the desired class [soil, vegetation]
 * mask always building and open surface water and forest 
 * ***************************************************************************/ 
 

var lulc= ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1')
            .filterDate(now.advance(-365,'day'), now)
            .filterBounds(region)
            .select('label')
            .reduce(ee.Reducer.mode())
            .rename('classification')
             
             
Map.addLayer(lulc.clip(region), dwVisParams, 'Classified Composite', false)

var masks=lulc.neq(0)
Map.addLayer(masks.clip(region).selfMask(),{min:1, max:1, palette:'white'},'mask images')


var boxcar = ee.Kernel.circle(10);
var collection = ee.ImageCollection("COPERNICUS/S1_GRD")
                  .filterBounds(region)
                  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
                  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VH'))
                  .filter(ee.Filter.eq('orbitProperties_pass', 'DESCENDING'))
                  .map(function(image){
                    return image.clip(region)
                                .select(['VV'])
                                .convolve(boxcar)
                  })
                  
                  
var soilmoisture = collection
                    .select(['VV'])
                    .filterDate(now.advance(-365, 'day'), now)
                    .map(function(image){
                      return image.updateMask(masks).multiply(slope)
                                  .add(intercept)
                                  .select('VV')
                                  .rename('SSM')
                                  .copyProperties(image, ['system:time_start'])
                    })
                    
                    
Map.addLayer(soilmoisture.mean(),{},'Soil moisture')


/***** Unspervised Clustering of Similar Pixels for better vis *******/

var radar_training=soilmoisture.mean().sample({
  region:region,
  scale:10,
  numPixels:5000, 
  seed:123,
})

// start with default of CascadeKmeans Clustering
var kmeans=ee.Clusterer.wekaCascadeKMeans(2,3).train(radar_training) 
var results=soilmoisture.mean().cluster(kmeans)

Map.addLayer(results,{min:0, max:2, palette:['red','yellow','blue']},'Clustering Soil moisture',true)
Map.addLayer(region.draw('white',3,1),{},'regions-aoi')
Map.centerObject(region,12)

```



![Workflow soil moisture extraction](https://github.com/mmbaye/Bolivia-example/blob/main/figures/Workflow.jpeg)









