<template>
  <div>
    <div>
      <div
        id="map"
        style="
          width: 100%;
          height: 100%;
          position: absolute;
          background-color: whitesmoke;
        "
      ></div>
      <!-- <Navbar /> -->
      <div class="container">
        <div class="button">
          <n-space vertical style="width: 200px" round type="primary">
            <n-select
              placeholder="Find Specialist or Clinic"
              filterable
              :options="spesialisListValue"
              @update:value="findSpecialist"
            />
          </n-space>
        </div>
        <div class="button">
          <n-space vertical style="width: 200px" round type="primary">
            <n-select
              placeholder="Find Hospital"
              filterable
              :options="rumahSakitListValue"
              @update:value="findRS"
            />
          </n-space>
        </div>
        <div class="button">
          <n-space vertical style="width: 200px" round type="primary">
            <n-select
              placeholder="Closest Hospital"
              filterable
              :options="spesialisListValue"
              @update:value="closestRS"
            />
          </n-space>
        </div>
      </div>
      <div class="TableData">
        <!-- <DataTable :value="rs_coordinate">
          <Column field="rs" header="Rumah Sakit"> </Column>
          <Column field="coordinates" header="Jarak">
            <template #body="slotProps">
              {{ slotProps.data.coordinates }}
            </template>
          </Column>
        </DataTable> -->
      </div>
    </div>
  </div>
</template>

<script>
import axios from "axios";
import Map from "ol/Map";
import Feature from "ol/Feature";
import View from "ol/View";
import TileLayer from "ol/layer/Tile";
import { BingMaps, Vector } from "ol/source";
import { getDistance } from "ol/sphere";
import VectorSource from "ol/source/Vector";
import { Vector as VectorLayer } from "ol/layer";
import {
  FeatureService,
  GetFeaturesBySQLParameters,
  TileSuperMapRest,
  TransportationAnalystResultSetting,
  TransportationAnalystParameter,
  FindClosestFacilitiesParameters,
  NetworkAnalystService,
} from "@supermap/iclient-ol";
import { GeoJSON } from "ol/format";
import { Icon, Style, Stroke } from "ol/style";
import { onMounted, ref, shallowRef, onUpdated } from "vue";
import { NButton, NDropdown, NCard, NSelect, NSpace } from "naive-ui";
import Point from "ol/geom/Point";

import Navbar from "../components/Navbar.vue";

export default {
  components: {
    NButton,
    NDropdown,
    NCard,
    NSelect,
    NSpace,
    Navbar,
  },
  setup() {
    let featureRS;
    let rs_coordinate;
    let userCoordinates;

    const radius = 6371008.8;
    const map = shallowRef();
    const spesialisList = ref([]);
    const spesialisListValue = ref([]);
    const rumahSakitListValue = ref([]);

    const successCallback = (position) => {
      const [x, y] = [position.coords.latitude, position.coords.longitude];
      userCoordinates = [x, y];
    };

    const errorCallback = (error) => {
      console.log(position);
    };
    const dataKoor = navigator.geolocation.getCurrentPosition(
      successCallback,
      errorCallback
    );
    let resultLayer = new VectorLayer({
      source: new VectorSource(),
      style: new Style({
        image: new Icon({
          src: "/rs.png",
        }),
      }),
    });

    let pathLayer = new VectorLayer({
      source: new Vector(),
      style: new Style({
        stroke: new Stroke({
          color: "rgba(22, 160, 133, 1)",
          width: 7.5,
        }),
      }),
    });
    let pathLayer2 = new VectorLayer({
      source: new Vector(),
      style: new Style({
        stroke: new Stroke({
          color: "rgba(63, 195, 128, 1)",
          width: 3.75,
        }),
      }),
    });

    async function findSpecialist(value) {
      rs_coordinate = [
        {
          rs: "RSND",
          coordinates: 50,
        },
        {
          rs: "RS JIWA",
          coordinates: 100,
        },
      ];

      map.value.removeLayer(pathLayer);
      map.value.removeLayer(pathLayer2);
      const res = await axios.get(
        `https://perfect-seal-belt.cyclic.app/spesialis/${value}`
      );
      const selected = res.data;

      const rsid = selected.data.map((r) => r.smid);
      let length = rsid.length;

      const features = featureRS.features.filter((feature) => {
        const smid = feature.properties["SMID"];
        return rsid.includes(parseInt(smid));
      });

      function convertFeatures(data) {
        let output = [];
        let i = 0;
        let obj = {};
        for (i = 0; i < data.length; i++) {
          obj = {};
          obj["rs"] = data[i].properties.NAME;
          obj["jarak"] = getDistance(
            [data[i].geometry.coordinates[1], data[i].geometry.coordinates[0]],
            [userCoordinates[0], userCoordinates[1]],
            radius
          );

          output.push(obj);
        }
        return output;
      }

      rs_coordinate = convertFeatures(features);

      console.log(rs_coordinate); // RS DAN JARAK LIST

      resultLayer.getSource().clear();
      resultLayer.getSource().addFeatures(
        new GeoJSON().readFeatures({
          type: "FeatureCollection",
          features: features,
        })
      );
      const extent = resultLayer.getSource().getExtent();
      // console.log(resultLayer);

      if (length == 1) {
        const centerPoint = [
          (extent[0] + extent[2]) / 2,
          (extent[1] + extent[3]) / 2,
        ];
        map.value.getView().animate({
          zoom: 17,
          center: centerPoint,
          duration: 500,
        });
      } else {
        map.value.getView().fit(extent, { duration: 500 });
      }
    }
    async function findRS(value) {
      map.value.removeLayer(pathLayer);
      map.value.removeLayer(pathLayer2);

      const res = await axios.get(
        `https://perfect-seal-belt.cyclic.app/rumahsakit/findRS/${value}`
      );
      const selected = res.data;

      const rsid = selected.data.map((r) => r.smid);

      const features = featureRS.features.filter((feature) => {
        const smid = feature.properties["SMID"];
        return rsid.includes(parseInt(smid));
      });

      resultLayer.getSource().clear();
      resultLayer.getSource().addFeatures(
        new GeoJSON().readFeatures({
          type: "FeatureCollection",
          features: features,
        })
      );

      const extent = resultLayer.getSource().getExtent();
      const centerPoint = [
        (extent[0] + extent[2]) / 2,
        (extent[1] + extent[3]) / 2,
      ];
      // map.value.getView().fit(extent, { duration: 500 });
      map.value.getView().animate({
        zoom: 17,
        center: centerPoint,
        duration: 500,
      });
    }
    async function closestRS(value) {
      map.value.removeLayer(pathLayer);
      map.value.removeLayer(pathLayer2);
      const res = await axios.get(
        `https://perfect-seal-belt.cyclic.app/spesialis/${value}`
      );
      const selected = res.data;

      const rsid = selected.data.map((r) => r.smid);
      const features = featureRS.features.filter((feature) => {
        const smid = feature.properties["SMID"];
        return rsid.includes(parseInt(smid));
      });

      const RSIcon = new Style({
        image: new Icon({
          src: "/RSv2.png",
        }),
      });
      resultLayer.getSource().clear();
      resultLayer.getSource().addFeatures(
        new GeoJSON().readFeatures({
          type: "FeatureCollection",
          features: features,
        })
      );

      // console.log(features);

      resultLayer.setStyle(RSIcon);

      const eventPoint = new Point([userCoordinates[1], userCoordinates[0]]);
      const eventPointString = {
        x: userCoordinates[1],
        y: userCoordinates[0],
      };

      const eventFeature = new Feature(eventPoint);
      // console.log(eventFeature);

      const eventIcon = new Style({
        image: new Icon({
          src: "/User.png",
        }),
      });

      const eventSource = new Vector({
        features: [eventFeature],
      });
      // console.log(eventSource);

      const eventLayer = new VectorLayer({
        source: eventSource,
      });

      eventLayer.setStyle(eventIcon);

      const resultSetting = new TransportationAnalystResultSetting({
        returnEdgeFeatures: true,
        returnEdgeGeometry: true,
        returnEdgeIDs: true,
        returnNodeFeatures: true,
        returnNodeGeometry: true,
        returnNodeIDs: true,
        returnPathGuides: true,
        returnRoutes: true,
      });
      // console.log(resultSetting);

      const analystParameter = new TransportationAnalystParameter({
        resultSetting: resultSetting,
        weightFieldName: "SmLength",
      });

      const featuresGeom = features.map((r) => r.geometry);
      const featuresCoordinates = featuresGeom.map((r) => r.coordinates);

      function convertToArrayOfObjects(data) {
        let output = [];
        let i = 0;
        let obj = {};
        for (i = 0; i < data.length; i++) {
          obj = {};
          obj["x"] = data[i][0];
          obj["y"] = data[i][1];

          output.push(obj);
        }

        return output;
      }

      let featuresCoordinatesObject =
        convertToArrayOfObjects(featuresCoordinates); //FUNCTION

      const findClosestFacilitiesParameters =
        new FindClosestFacilitiesParameters({
          event: eventPointString,
          expectFacilityCount: 1,
          facilities: featuresCoordinatesObject,
          isAnalyzeById: false,
          parameter: analystParameter,
        });
      // console.log(findClosestFacilitiesParameters);

      new NetworkAnalystService(
        "https://iserver.supermap.id/iserver/services/transportationAnalyst-SpatialDataWebGISRS/rest/networkanalyst/Data_WebGIS_Network@Data_WebGIS"
      ).findClosestFacilities(
        findClosestFacilitiesParameters,
        function (serviceResult) {
          // console.log(serviceResult);
          serviceResult.result.facilityPathList.map(function (result) {
            // console.log(result);

            pathLayer.getSource().clear();
            // console.log(pathLayer.getSource());
            pathLayer
              .getSource()
              .addFeatures(new GeoJSON().readFeatures(result.route));
            pathLayer2.getSource().clear();
            // console.log(pathLayer.getSource());
            pathLayer2
              .getSource()
              .addFeatures(new GeoJSON().readFeatures(result.route));

            // console.log(pathLayer.getSource());
            const pathLayerExtent = pathLayer.getSource().getExtent();
            map.value.getView().fit(pathLayerExtent, { duration: 500 });

            map.value.removeLayer(pathLayer);
            map.value.removeLayer(pathLayer2);
            map.value.addLayer(pathLayer);
            map.value.addLayer(pathLayer2);
          });
        }
      );
      map.value.addLayer(eventLayer);
    }
    onMounted(() => {
      axios
        .get("https://perfect-seal-belt.cyclic.app/spesialislabelkey")
        .then((result) => {
          spesialisList.value = result.data.spesialislabelkey;
        });

      axios
        .get("https://perfect-seal-belt.cyclic.app/spesialislabelvalue")
        .then((result) => {
          spesialisListValue.value = result.data.spesialislabelvalue;
        });
      axios
        .get("https://perfect-seal-belt.cyclic.app/rumahsakit")
        .then((result) => {
          rumahSakitListValue.value = result.data.rumahsakit;
        });
      const url =
        "https://iserver.supermap.id/iserver/services/map-SpatialDataWebGISRS/rest/maps/SpatialDataWebGISRS";

      map.value = new Map({
        target: "map",
        view: new View({
          center: [110.42, -6.99],
          zoom: 13,
          projection: "EPSG:4326",
        }),
      });

      let layer = new TileLayer({
        source: new TileSuperMapRest({
          url: url,
          wrapX: true,
        }),
        projection: "EPSG:4326",
      });

      let bingMapLayer = new TileLayer({
        visible: true,
        preload: Infinity,
        source: new BingMaps({
          key: "Aug_jpyeictKv9-blHjr0OJOy-hYRR_5bIWhecGYlywC_F6p0LMvQ0Ye8J95tSJt",
          imagerySet: "Road",
        }),
      });
      map.value.addLayer(bingMapLayer);
      map.value.addLayer(resultLayer);

      const sqlParam = new GetFeaturesBySQLParameters({
        toIndex: 26,
        queryParameter: {
          name: "RS@Data_WebGIS",
          attributeFilter: "1 = 1",
        },
        datasetNames: ["Data_WebGIS:RS"],
      });

      new FeatureService(
        "https://iserver.supermap.id/iserver/services/data-SpatialDataWebGISRS-3/rest/data"
      ).getFeaturesBySQL(sqlParam, function (serviceResult) {
        // console.log(serviceResult);
        resultLayer
          .getSource()
          .addFeatures(
            new GeoJSON().readFeatures(serviceResult.result.features)
          );
        featureRS = serviceResult.result.features;
        // console.log(featureRS);
      });
    });
    return {
      spesialisList,
      spesialisListValue,
      rumahSakitListValue,
      featureRS,
      rs_coordinate,
      findRS,
      closestRS,
      findSpecialist,
    };
  },
};
</script>

<style>
@import "ol/ol.css";
.right {
  position: absolute;
  float: right;
  text-align: center;
}
.containerWeb {
  width: 500px;
  height: 250px;
  margin: 50px;

  display: flex;
  justify-content: center;
}
.queryForm {
  position: absolute;
  float: center;
}
.container {
  width: 75%;
  display: flex;
  justify-content: center;
  position: relative;

  top: 25px;
}
.button {
  margin: 5px;
}
.TableData {
  width: 75%;
  display: flex;
  justify-content: center;
  position: relative;

  top: 300px;
}
</style>
