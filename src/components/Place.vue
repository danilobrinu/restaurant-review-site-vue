<template>
  <div
    role="button"
    tabIndex="0"
    class="flex p-3 -mx-3 my-2 rounded hover:shadow-md focus:shadow-md"
    @keydown.prevent.enter.space="handleClick"
    @click.prevent="handleClick"
  >
    <div class="w-32 min-w-32 h-32">
      <cover
        :source="cover"
        :fallback="`https://www.toptal.com/designers/subtlepatterns/patterns/food.png`"
        class="w-32 h-32 object-cover bg-gray-200 rounded"
        alt="cover"
      />
    </div>
    <div class="flex flex-col items-start flex-1 pl-4">
      <div class="flex items-start mb-2">
        <RatingBadge class="mr-2">{{ rating > 0 ? rating.toFixed(1) : 'New' }}</RatingBadge>
        <span class="inline-block text-lg font-bold leading-none text-gray-900">
          {{ name }}
        </span>
      </div>
      <div>
        <rating-type-badge v-for="type in types" :key="type">{{ type }}</rating-type-badge>
      </div>
      <div class="text-xs font-bold text-gray-600 mt-auto">
        {{ ratings > 0 ? `${ratings} ratings` : 'Without ratings' }}
      </div>
    </div>
  </div>
</template>

<script>
////////////////////////////////////////////////////////////////////////////////////////////////////

import RatingBadge from './RatingBadge.vue';
import RatingTypeBadge from './RestaurantTypeBadge.vue';
import Cover from './Cover.vue';

////////////////////////////////////////////////////////////////////////////////////////////////////

export default {
  name: 'Place',
  props: ['name', 'cover', 'rating', 'ratings', 'types'],
  setup(_, { emit }) {
    let handleClick = () => {
      emit('handle-click');
    };

    return { handleClick };
  },
  components: { RatingBadge, RatingTypeBadge, Cover },
};
</script>

<style></style>
