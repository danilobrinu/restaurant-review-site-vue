<template>
  <div class="absolute inset-0">
    <div class="flex relative w-full h-full overflow-hidden">
      <div class="flex-initial relative w-full max-w-xl h-full shadow-xl z-50">
        <div class="flex flex-col w-full h-full">
          <header class="flex-none">
            <nav class="flex py-4 border-b">
              <div class="w-full px-6">
                <div class="flex items-center h-12 shadow-md rounded">
                  <label for="" class="w-full">
                    <div class="flex">
                      <div
                        class="flex flex-initial w-10 h-10 items-center justify-center cursor-pointer"
                      >
                        <fa-icon class="text-gray-900" icon="search"></fa-icon>
                      </div>
                      <input
                        type="text"
                        class="flex-auto h-10 font-bold text-gray-900 placeholder-gray-600 rounded"
                        placeholder="Search a Restaurant"
                        v-model.trim="state.query"
                      />
                      <div
                        v-if="state.query.length > 0"
                        class="flex flex-initial w-10 h-10 items-center justify-center cursor-pointer"
                        tab-index="0"
                        role="button"
                        @keydown.prevent.enter.space="state.query = ''"
                        @click="state.query = ''"
                      >
                        <fa-icon class="text-gray-900" icon="times"></fa-icon>
                      </div>
                    </div>
                  </label>
                </div>
                <div class="flex mt-3">
                  <div class="flex-1">
                    <select
                      class="w-full h-12 px-4 font-bold text-gray-900 bg-gray-200 shadow-inner appeareance-none rounded"
                      v-model="state.minRating"
                    >
                      <option value="0">New</option>
                      <option
                        v-for="option in state.minRatingRange"
                        :key="`min-rating-option-${option}`"
                        :value="option"
                        >{{ option }}</option
                      >
                    </select>
                  </div>
                  <div class="flex items-center justify-center w-12">
                    <span class="font-bold">⯈</span>
                  </div>
                  <div class="flex-1">
                    <select
                      class="w-full h-12 px-4 font-bold text-gray-900 bg-gray-200 shadow-inner appeareance-none rounded"
                      v-model="state.maxRating"
                    >
                      <option
                        v-for="option in state.maxRatingRange"
                        :key="`min-rating-option-${option}`"
                        :value="option"
                        >{{ option }}</option
                      >
                    </select>
                  </div>
                </div>
              </div>
            </nav>
          </header>
          <div class="relative flex-1 max-h-full">
            <div class="absolute inset-0">
              <div class="relative w-full h-full max-h-full overflow-x-hidden overflow-y-scroll">
                <main class="p-6" @keydown.prevent="handleKeyboardNavigationPlaces">
                  <div v-if="state.placesFiltered.length > 0">
                    <place
                      v-for="restaurant in state.placesFiltered"
                      :key="restaurant.id"
                      :id="restaurant.id"
                      :name="restaurant.name"
                      :cover="restaurant.cover"
                      :types="restaurant.types"
                      :rating="restaurant.rating"
                      :ratings="restaurant.ratings"
                      @handle-click="state.placeId = restaurant.id"
                    ></place>
                  </div>
                  <div v-else>
                    <div class="flex items-center justify-center mb-6">
                      <fa-icon class="text-purple-400" icon="sad-tear" size="5x"></fa-icon>
                    </div>
                    <div class="text-2xl font-bold text-gray-900 text-center">
                      No Results!
                    </div>
                    <div class="text-gray-700 text-center">
                      Sorry there are no results for this search. Please try again.
                    </div>
                  </div>
                </main>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div v-if="state.showPlaceDetailsModal">
        <query
          :query="getPlaceDetailsQuery"
          :variables="{ service: state.service, placeId: state.placeId }"
          :sync="props => () => props.variables.placeId"
        >
          <template slot-scope="{ loading, data }">
            <div v-if="loading" class="absolute left-xl w-full max-w-lg h-screen bg-white z-10">
              <scrollable :trigger="state.placeId">
                <place-details-placeholder></place-details-placeholder>
              </scrollable>
            </div>

            <div
              v-else
              role="dialog"
              tabIndex="0"
              class="absolute left-xl w-full max-w-lg h-screen bg-white z-10"
            >
              <scrollable :trigger="state.placeId">
                <place-details
                  :place="data"
                  @handle-click-add-review="state.showAddReviewModal = true"
                  @handle-close="state.placeId = null"
                ></place-details>
              </scrollable>
            </div>

            <div
              v-if="state.showAddReviewModal"
              class="absolute left-xl w-full max-w-lg h-screen z-20"
            >
              <div class="absolute inset-0 bg-black opacity-50 -z-1" />

              <div class="p-6">
                <add-review-form
                  @handle-submit="review => addNewReview(review, data)"
                  @handle-cancel="state.showAddReviewModal = false"
                ></add-review-form>
              </div>
            </div>
          </template>
        </query>
      </div>

      <div
        v-if="state.showAddRestaurantModal"
        class="absolute left-xl w-full max-w-lg h-screen z-20"
      >
        <div class="absolute inset-0 bg-black opacity-50 -z-1" />
        <div class="p-6">
          <add-restaurant-form
            :location="state.locationClicked"
            @handle-submit="() => false"
            @handle-cancel="state.showAddRestaurantModal = false"
          ></add-restaurant-form>
        </div>
      </div>

      <div class="flex-auto relative">
        <div ref="mapContainer" class="absolute inset-0"></div>
      </div>
    </div>
  </div>
</template>

<script>
import { reactive, ref, computed, watch, onMounted } from '@vue/composition-api';
import {
  range,
  debounce,
  getCurrentPosition,
  getNearbyPlaces,
  getSortedPlaces,
  getFilteredPlaces,
  getPlaceDetails,
  cleanMarkers,
  noop,
} from './helpers';

////////////////////////////////////////////////////////////////////////////////////////////////////

import Place from './components/Place.vue';
import AddRestaurantForm from './components/AddRestaurantForm.vue';
import AddReviewForm from './components/AddReviewForm.vue';
import PlaceDetails from './components/PlaceDetails.vue';
import PlaceDetailsPlaceholder from './components/PlaceDetailsPlaceholder.vue';
import Scrollable from './components/Scrollable.vue';
import Query from './components/Query.vue';

////////////////////////////////////////////////////////////////////////////////////////////////////

window.places = {};
window.markers = {};

export default {
  name: 'app',
  setup() {
    let mapContainer = ref(null);
    let state = reactive({
      query: '',
      minRatingRange: range(1, 5),
      minRating: 0,
      maxRatingRange: computed(() => range(state.minRating + 1, 6)),
      maxRating: 5,
      map: null,
      service: null,
      locationClicked: { lat: 0, lng: 0 },
      showAddRestaurantModal: false,
      showAddReviewModal: false,
      centerPosition: new window.google.maps.LatLng(0, 0),
      places: {},
      placesFiltered: computed(() =>
        getSortedPlaces(
          getFilteredPlaces(state.places, state.query, state.minRating, state.maxRating)
        )
      ),
      placeId: null,
      showPlaceDetailsModal: computed(() => !!state.placeId),
    });

    const getPlaceDetailsQuery = ({ service, placeId }) => getPlaceDetails(service, placeId);

    onMounted(() => {
      state.map = new window.google.maps.Map(mapContainer.value, {
        center: new window.google.maps.LatLng(0, 0),
        zoom: 16,
        fullscreenControl: false,
        mapTypeControl: false,
        gestureHandling: 'cooperative',
      });
      state.service = new window.google.maps.places.PlacesService(state.map);

      state.map.addListener('click', e => {
        state.locationClicked = e.latLng.toJSON();
        state.showAddRestaurantModal = true;
      });

      state.map.addListener(
        'bounds_changed',
        debounce(() => {
          state.centerPosition = state.map.getCenter();
        })
      );

      getCurrentPosition()
        .then(position => {
          state.centerPosition = position;
          state.map.setCenter(position);

          new window.google.maps.Marker({
            map: state.map,
            position,
            icon: {
              url: 'images/marker.png',
              scaledSize: { width: 32, height: 48 },
              labelOrigin: { x: 16, y: 18 },
            },
            label: '😊',
          });
        })
        .catch(noop);
    });

    watch(
      () => state.centerPosition,
      () => {
        getNearbyPlaces(state.service, state.centerPosition)
          .then(result => {
            state.places = result;
          })
          .catch(noop);
      }
    );

    watch(
      () => state.placesFiltered,
      () => {
        cleanMarkers();

        const markers = Object.values(state.placesFiltered).reduce(
          (accumulator, { location: position, id, rating }) => {
            const marker = new window.google.maps.Marker({
              map: state.map,
              position,
              icon: {
                url: 'images/marker.png',
                scaledSize: { width: 32, height: 48 },
                labelOrigin: { x: 16, y: 18 },
              },
              label: {
                fontFamily: '"Montserrat", sans-serif',
                fontWeight: '600',
                fontSize: '12px',
                color: '#fff',
                text: rating > 0 ? rating.toFixed(1) : 'N',
              },
            });

            marker.addListener('click', () => {
              state.placeId = id;
            });

            accumulator[id] = marker;

            return accumulator;
          },
          {}
        );

        window.markers = markers;
      }
    );

    watch(
      () => state.places,
      () => {
        if (state.placeId && !Object.prototype.hasOwnProperty.call(state.places, state.placeId)) {
          state.placeId = null;
        }
      }
    );

    let addNewRestaurant = data => {
      window.places[data.id] = data;

      state.places[data.id] = window.places[data.id];
      state.showAddRestaurantModal = false;
    };

    let addNewReview = (data, place) => {
      const reviews = [data, ...place.reviews];
      const rating = place.rating
        ? // Incremental Average:
          // Formula: ( ( ( Total Count - 1 ) * Previous Rating (AVG) ) + New Rating ) / Total Count
          (place.ratings * place.rating + data.rating) / (place.ratings + 1)
        : data.rating;
      const ratings = place.ratings + 1;

      window.places[place.id] = {
        ...place,
        rating,
        ratings,
        reviews,
      };
      window.markers[place.id].setLabel(rating.toString());

      state.places[place.id] = window.places[place.id];
      state.showAddReviewModal = false;
    };

    const handleKeyboardNavigationPlaces = e => {
      e.preventDefault();

      const next = e.target.nextSibling;
      const previous = e.target.previousSibling;

      switch (e.key) {
        case 'ArrowUp':
        case 'ArrowLeft':
          if (previous) {
            previous.focus();
          }
          break;
        case 'ArrowDown':
        case 'ArrowRight':
          if (next) {
            next.focus();
          }
          break;
        default:
          break;
      }
    };

    return {
      state,
      mapContainer,
      addNewRestaurant,
      addNewReview,
      handleKeyboardNavigationPlaces,
      getPlaceDetailsQuery,
    };
  },
  components: {
    Place,
    AddRestaurantForm,
    AddReviewForm,
    PlaceDetails,
    PlaceDetailsPlaceholder,
    Query,
    Scrollable,
  },
};
</script>

<style>
body {
  font-family: 'Montserrat', sans-serif;
}
.left-xl {
  left: 36rem;
}
.-z-1 {
  z-index: -1;
}
.max-w-lg {
  max-width: 32rem;
}
</style>
