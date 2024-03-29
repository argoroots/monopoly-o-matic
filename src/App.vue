<script setup>
import { computed, ref, watch } from 'vue'

import ImageComponent from '@/components/ImageComponent.vue'

import PropertyList from '@/components/PropertyList.vue'
import PropertyCard from '@/components/PropertyCard.vue'
import PlayerList from '@/components/PlayerList.vue'
import PlayerListEdit from '@/components/PlayerListEdit.vue'

import { allProperties } from '@/data/properties.js'
import { allPlayers } from '@/data/players.js'
import { allCards } from '@/data/cards.js'

const properties = ref(getData('properties') || allProperties.map((p, idx) => ({ id: idx, ...p })))
const players = ref(getData('players') || allPlayers.map((p, idx) => ({ id: idx, ...p })))
const cards = ref(getData('cards') || allCards.map((c, idx) => ({ id: idx, ...c })))

const propertyId = ref()
const playerId = ref(1)
const editPlayers = ref(false)

const selectedPlayers = computed(() => players.value.filter(p => p.selected))

const property = computed(() => properties.value.find(p => p.id === propertyId.value))

const player = computed(() => players.value.find(p => p.id === playerId.value))

const sameGroupProperties = computed(() => {
  if (!property.value) {
    return null
  }

  return properties.value.filter(p => p.group === property.value.group)
})

const avgHouses = computed(() => {
  if (!property.value) {
    return null
  }

  const houses = sameGroupProperties.value.map(p => p.houses)
  const sum = houses.reduce((a, b) => a + b, 0)
  return (sum / houses.length) || 0
})

const playerGroupProperties = computed(() => {
  if (!property.value) {
    return null
  }

  return sameGroupProperties.value.filter(p => p.owner !== null && p.owner === property.value.owner)
})

const playerOwnsGroup = computed(() => {
  return sameGroupProperties.value.length === playerGroupProperties.value.length
})

const rent = computed(() => {
  if (!property.value) {
    return null
  }

  if (property.value.type === 'property') {
    if (playerOwnsGroup.value) {
      return property.value.rent[property.value.houses] * 2
    } else {
      return property.value.rent[property.value.houses]
    }
  }

  if (property.value.type === 'station' || property.value.type === 'utility') {
    const idx = playerGroupProperties.value.length
    return property.value.rent[idx - 1]
  }

  return 0
})

const actions = computed(() => {
  const p = property.value
  const u = player.value
  const pu = players.value[p.owner]
  const result = []

  if (p.type === 'go') {
    result.push({
      label: `Pay £${p.price} salary to ${u.name}`,
      action: () => {
        u.balance += p.price
      }
    })
  }

  if (p.type === 'jail') {
    result.push({
      label: `Collect £${p.price} fine from ${u.name}`,
      action: () => {
        u.balance -= p.price
      }
    })
  }

  if (p.type === 'chance' || p.type === 'chest') {
    cards.value.filter(c => c.card === p.type).forEach(c => {
      let total = c.sum

      switch (c.type) {
        case 'repairs':
          total = properties.value.filter(p => p.owner === playerId.value && p.houses > 0 && p.houses < 5).map(p => p.houses).reduce((a, b) => a + b, 0) * c.sum.house + properties.value.filter(p => p.owner === playerId.value && p.houses === 5).map(p => 1).reduce((a, b) => a + b, 0) * c.sum.hotel
          break
        case 'birthday':
          total = c.sum * (players.value.length - 1) * -1
          break
      }

      if (total === 0) { return }

      result.push({
        label: c.label.replace('#SUM', Math.abs(c.sum)).replace('#TOTAL', Math.abs(total)).replace('#PLAYER', u.name),
        action: () => {
          switch (c.type) {
            case 'repairs':
              break

            case 'birthday':
              players.value.forEach(p => {
                if (p.id !== playerId.value) {
                  p.balance -= c.sum
                }
              })
              break
          }

          u.balance -= total
        }
      })
    })
  }

  if (p.type === 'tax') {
    result.push({
      label: `Collect £${p.price} tax from ${u.name}`,
      action: () => {
        u.balance -= p.price
      }
    })
  }

  if (['property', 'station', 'utility'].includes(p.type)) {
    if (p.owner === null) {
      result.push({
        disabled: u.balance < p.price,
        label: `Sell property to ${u.name} for £${p.price}`,
        action: () => {
          u.balance -= p.price
          p.owner = playerId.value
        }
      })
    }

    if (p.owner !== null && p.owner !== playerId.value) {
      const actualRent = p.type === 'utility' ? rent.value * Math.floor(Math.random() * 11) + 2 : rent.value

      result.push({
        disabled: p.mortgage,
        label: `Transfer £${actualRent} rent from ${u.name} to ${pu.name}`,
        action: () => {
          u.balance -= actualRent
          pu.balance += actualRent
        }
      })
    }

    if (p.owner !== null && !p.mortgage) {
      result.push({
        disabled: p.houses !== 0,
        label: `Mortgage property for £${p.price / 2}`,
        action: () => {
          pu.balance += p.price / 2
          p.mortgage = true
        }
      })
    }

    if (p.owner !== null && p.mortgage) {
      result.push({
        disabled: pu.balance < p.price / 2 * 1.1,
        label: `Lift mortgage for £${Math.round(p.price / 2 * 1.1)}`,
        action: () => {
          pu.balance -= Math.round(p.price / 2 * 1.1)
          p.mortgage = false
        }
      })
    }
  }

  if (p.type === 'property') {
    if (avgHouses.value >= p.houses && p.houses < 5 && p.owner !== null && !p.mortgage && playerOwnsGroup.value) {
      result.push({
        disabled: pu.balance < p.housePrice,
        label: p.houses < 4 ? `Sell house #${p.houses + 1} for £${p.housePrice}` : `Sell hotel for £${p.housePrice}`,
        action: () => {
          pu.balance -= p.housePrice
          p.houses += 1
        }
      })
    }

    if (avgHouses.value <= p.houses && p.houses > 0 && p.owner !== null && !p.mortgage && playerOwnsGroup.value) {
      result.push({
        label: p.houses < 5 ? `Buy back house #${p.houses} for £${p.housePrice / 2}` : `Buy back hotel for £${p.housePrice / 2}`,
        action: () => {
          pu.balance += p.housePrice / 2
          p.houses -= 1
        }
      })
    }
  }

  return result
})

watch(properties.value, () => {
  setData('properties', properties.value)
})

watch(players.value, () => {
  setData('players', players.value)
})

function onReset () {
  setData('properties')
  setData('players')

  location.reload()
}

function getData (key) {
  const data = localStorage.getItem(key)

  if (data) {
    return JSON.parse(data)
  }
}

function setData (key, data) {
  if (data) {
    localStorage.setItem(key, JSON.stringify(data))
  } else {
    localStorage.removeItem(key)
  }
}
</script>

<template>
  <transition>
    <div
      v-if="editPlayers"
      class="players-edit"
    >
      <player-list-edit
        v-model="players"
        @reset="onReset"
        @close="editPlayers = false"
      />
    </div>
  </transition>

  <div class="board">
    <property-list
      v-model="propertyId"
      :properties="properties"
    />

    <div class="players">
      <player-list
        v-model="playerId"
        :players="selectedPlayers"
      />
      <button
        class="mt-2 py-1 px-2 pr-3 box-content flex flex-row justify-between items-center text-sm text-stone-900/70 border border-transparent hover:border-stone-900/20 rounded-sm active:bg-emerald-50"
        @click="editPlayers = true"
      >
        <image-component
          src="edit.png"
          class="h-3 w-3 mr-2"
        /> Edit
      </button>
    </div>

    <div class="center">
      <image-component
        src="monopoly.svg"
        class="h-52 w-52"
      />

      <h1 class="w-max text-2xl text-center font-bold text-stone-900/80 tracking-wide">
        Monopoly-O-Matic
        <span class="block italic text-right text-sm font-thin">by <a
          class="hover:underline"
          href="mailto:argo@roots.ee?subject=Monopoly-O-Matic"
        >Argo Roots</a></span>
      </h1>
    </div>

    <div class="card">
      <transition>
        <property-card
          v-if="property"
          :property="property"
          :player="player"
          :actions="actions"
        />
      </transition>
    </div>
  </div>
</template>

<style>
.board {
  @apply h-full grid justify-center content-center border border-white bg-emerald-100;
  grid-template-columns: repeat(11, minmax(0, 1fr));
  grid-template-rows: repeat(11, minmax(0, 1fr));
}

.players {
  @apply p-8 flex flex-col justify-center items-center border-white;
  grid-row: span 9 / span 9;
  grid-column: span 3 / span 3;
}

.players-edit {
  @apply absolute inset-0;
}

.center {
  @apply p-8 flex flex-col justify-center items-center border-white;
  grid-row: span 9 / span 9;
  grid-column: span 3 / span 3;
}

.card {
  @apply p-8 flex flex-col justify-center items-center border-white;
  grid-row: span 9 / span 9;
  grid-column: span 3 / span 3;
}

.v-enter-active,
.v-leave-active {
  transition: opacity 0.5s ease;
}

.v-enter-from,
.v-leave-to {
  opacity: 0;
}
</style>
