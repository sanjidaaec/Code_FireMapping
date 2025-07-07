# Code_FireMapping
var fireloc = ee.Geometry.Point([-122.90, 41.84]);
var prefire = imageCollection.filterDate('2022-07-01','2022-07-28')
.filterBounds(fireloc)
.filterMetadata('CLOUD_COVER', 'less_than', 10)
.median()
.multiply(0.0000275).add(-0.2);
print(prefire);

Map.addLayer(prefire, {bands: ['SR_B7', 'SR_B5', 'SR_B4'], max: 0.3}, 'Landsat 8 Image prefire');

//Step2: repeat the above to include the postfire dates.
var postfire = imageCollection.filterDate('2022-07-29', '2022-08-28')
.filterBounds(fireloc)
.filterMetadata('CLOUD_COVER', 'less_than', 10)
.median().multiply(0.0000275).add(-0.2);
print (postfire);

Map.addLayer(postfire, {bands: ['SR_B7', 'SR_B5', 'SR_B4'], max: 0.3}, 'Landsat 8 Image postfire' );

// NBR = (NIR - SWIR2)/ (NIR+SWIR2)
//dNBR = NBRprefire - NBR postfire

var NBRprefire = prefire.normalizedDifference(['SR_B5', 'SR_B7']);
var NBRpostfire = postfire.normalizedDifference(['SR_B5', 'SR_B7']);

var dNBR = NBRprefire.subtract(NBRpostfire);
Map.addLayer(NBRprefire,{min: -1, max: 1, palette: ['blue','white','yellow']}, 'Prefire NBR');
Map.addLayer(NBRpostfire,{min: -1, max: 1, palette: ['blue','white','yellow']}, 'Postfire NBR');
Map.addLayer(dNBR,{min: -1, max: 1, palette: ['green','white','red']}, 'McKinnsey dNBR');

//clip dNBR to aoi

var dNBRclip = dNBR.clip(aoi);

//Define the burn severity thresholds
var EnhancedRegrowth= -0.1;
var Unburned = 0.1;
var LowSeverity = 0.27
var ModerateSeverity = 0.66;
var HighSeverity = 3;

//Perform burn severity classification using thresholding
var burnSeverity = dNBRclip.expression(
  'i <= Regrowth ? 0 : (i <= Unburned ? 1 : (i <= Low ? 2 : (i <= Moderate ? 3 : (i <= High ? 4 : 5))))', {
    'i': dNBR,
    'Regrowth' : EnhancedRegrowth,
    'Unburned' : Unburned,
    'Low': LowSeverity,
    'Moderate' : ModerateSeverity,
    'High' : HighSeverity
  }).int().clip(aoi);
  
// Define the burn severity class colors  
  var classPalette = ['grey', 'green', 'yellow', 'orange', 'red'];
  
  // Display the classified burn severity map
  
  Map.addLayer(burnSeverity, {min: 0, max: 4, palette: classPalette}, 'Burn Severity');
  
  print('Burn Severity Classes');
  print('Class 0: Enhanced Regrowth');
  print('Class 1: Unburned');
  print('Class 2: Low Severity');
  print('Class 3: Moderate Severity');
  print('Class4: High Severity');
  
  //Filter the image to extract classes 3 and 4
  
  var BurntClasses = burnSeverity.gt(1).and(burnSeverity.lt(5));
  Map.addLayer(BurntClasses, {min: 0, max: 1, palette: classPalette}, 'Burned');
  print(BurntClasses);
  
  //this allow us to see that the name of the band is 'consatant'
  
  var areaImage = BurntClasses.multiply(ee.Image.pixelArea());
  var area = areaImage.reduceRegion({
    reducer: ee.Reducer.sum(),
    geometry:dNBRclip.geometry(),
    scale: 30,
    maxPixels: 1e10
  });
  var BurntAreaSqKm = ee.Number(
    area.get('constant')).divide(1e6).round();
    
    print(BurntAreaSqKm)
    
//.........................
//CREATE A LEGEND
//........................
//Create a Legend

var legend = ui.Panel({
  style: {
    position: 'bottom-right',
    padding: '8px',
    width: '200px'
  }
});

//Create the legend title

var legendTitle = ui.Label({
  value: 'Burn Severity Legend',
  style: {fontWeight: 'bold', fontSize: '18px'}
});

//Add the title to the Legend
legend.add(legendTitle);

//Create and add the Legend entries 
var entry0 = ui.Label({value: 'Class 0: Enhanced Regrowth', style: {color: classPalette[0]}});
var entry1 = ui.Label({value: 'Class 1: Unburned', style: {color: classPalette[1]}});
var entry2 = ui.Label({value: 'Class 2: Low Severity', style: {color: classPalette[2]}});
var entry3 = ui.Label({value: 'Class 3: Moderate Severity', style: {color: classPalette[3]}});
var entry4 = ui.Label({value: 'Class 4: High Severity', style: {color: classPalette[4]}});

//Add the legend entries to the legend
legend.add(entry0).add(entry1).add(entry2).add(entry3).add(entry4);

//Add the legend to the map
Map.add(legend);

//Center the Map view on the AOI
Map.centerObject(fireloc, 10);

