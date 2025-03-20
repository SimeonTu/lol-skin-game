<template>
  <div class="game-container" @click="handleContainerClick">
    <div class="image-container">
      <div v-if="isLoading" class="loading-screen">
        Loading next champion...
      </div>
      <img 
        v-show="!isLoading"
        :src="currentSkin" 
        :style="zoomStyle" 
        :class="{ 'animate-zoom': isAnimating }" 
        alt="Champion skin"
        @load="handleImageLoad"
      />
      <div v-if="isCorrect" class="feedback-message correct">Correct! It was {{ currentChampion.name }}!</div>
      <div v-if="showIncorrectMessage" class="feedback-message incorrect">Incorrect! Try again...</div>
      <div class="attempts-counter">Attempts: {{ attempts }}</div>
    </div>
    
    <div class="input-container">
      <div class="autocomplete">
        <input
          ref="guessInput"
          v-model="guess"
          @input="filterChampions"
          @keyup.enter="submitGuess"
          @keydown.down.prevent="navigateList(1)"
          @keydown.up.prevent="navigateList(-1)"
          placeholder="Guess the champion..."
          :disabled="isCorrect || inputLocked"
        />
        <div v-if="showSuggestions && filteredChampions.length" class="suggestions">
          <div
            v-for="(champion, index) in filteredChampions"
            :key="champion.id"
            @click="selectChampion(champion)"
            :class="['suggestion-item', { 'selected': index === selectedIndex }]"
          >
            <div class="champion-wrapper">
              <img 
                :src="getChampionIconUrl(champion.id)"
                :alt="champion.name"
                class="champion-icon"
              />
              <span>{{ champion.name }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      champions: [],
      currentChampion: null,
      currentSkin: '',
      availableSkins: [],
      zoomLevel: 2,
      attempts: 0,
      guess: '',
      positionX: 0,
      positionY: 0,
      version: '13.24.1', // Add your current version
      isCorrect: false,
      showSuggestions: false,
      filteredChampions: [],
      isAnimating: false,
      isLoading: true,
      selectedIndex: -1,
      showIncorrectMessage: false,
      inputLocked: false,
      championIcons: {}, // Store preloaded icons
    };
  },
  computed: {
    zoomStyle() {
      return {
        transform: `scale(${this.zoomLevel}) translate(${this.positionX}%, ${this.positionY}%)`,
        transformOrigin: 'center',
      };
    },
  },
  methods: {
    async fetchChampionData() {
      try {
        const response = await fetch(
          `https://ddragon.leagueoflegends.com/cdn/${this.version}/data/en_US/champion.json`
        );
        const data = await response.json();
        this.champions = Object.values(data.data).map(champion => ({
          id: champion.id,
          name: champion.name,
        }));

        // Preload champion icons
        this.champions.forEach(champion => {
          const img = new Image();
          img.src = `https://ddragon.leagueoflegends.com/cdn/${this.version}/img/champion/${champion.id}.png`;
          this.championIcons[champion.id] = img.src;
        });

        await this.selectRandomChampion();
      } catch (error) {
        console.error('Error fetching champion data:', error);
      }
    },
    async selectRandomChampion() {
      this.isLoading = true;
      const randomChamp = this.champions[Math.floor(Math.random() * this.champions.length)];
      
      // Fetch champion details including skins
      const response = await fetch(
        `https://ddragon.leagueoflegends.com/cdn/${this.version}/data/en_US/champion/${randomChamp.id}.json`
      );
      const data = await response.json();
      const championData = data.data[randomChamp.id];
      
      this.currentChampion = randomChamp;
      this.availableSkins = championData.skins;
      
      // Select random skin
      const randomSkin = this.availableSkins[Math.floor(Math.random() * this.availableSkins.length)];
      this.currentSkin = `https://ddragon.leagueoflegends.com/cdn/img/champion/splash/${randomChamp.id}_${randomSkin.num}.jpg`;
      
      this.zoomLevel = 2;
      this.attempts = 0;
      this.isCorrect = false;
      this.guess = '';
      this.isAnimating = false;
      
      const maxOffset = 25;
      this.positionX = Math.random() * maxOffset * 2 - maxOffset;
      this.positionY = Math.random() * maxOffset * 2 - maxOffset;
      
      this.$nextTick(() => {
        this.focusInput();
      });
    },
    filterChampions() {
      if (!this.guess) {
        this.filteredChampions = [];
        this.showSuggestions = false;
        this.selectedIndex = -1;
        return;
      }
      
      const searchTerm = this.guess.toLowerCase();
      this.filteredChampions = this.champions.filter(champion =>
        champion.name.toLowerCase().includes(searchTerm)
      ).slice(0, 5);
      
      this.showSuggestions = true;
      this.selectedIndex = -1; // Reset selection when filtering
    },
    selectChampion(champion) {
      this.guess = champion.name;
      this.showSuggestions = false;
      this.checkGuess();
      this.focusInput();
    },
    checkGuess() {
      this.attempts++;
      if (this.guess.toLowerCase() === this.currentChampion.name.toLowerCase()) {
        this.isCorrect = true;
        this.isAnimating = true;
        // Reset zoom and position to show full centered image
        this.zoomLevel = 1;
        this.positionX = 0;
        this.positionY = 0;
        
        setTimeout(() => {
          this.selectRandomChampion();
        }, 2000);
      } else {
        this.showIncorrectMessage = true;
        this.inputLocked = true;
        this.isAnimating = true;
        this.zoomLevel = Math.max(1, this.zoomLevel - 0.25);
        const zoomProgress = (2 - this.zoomLevel) / 1;
        this.positionX *= (1 - zoomProgress);
        this.positionY *= (1 - zoomProgress);
        
        setTimeout(() => {
          this.showIncorrectMessage = false;
          this.inputLocked = false;
          this.focusInput();
        }, 500);
      }
    },
    submitGuess() {
      if (this.selectedIndex >= 0) {
        this.selectChampion(this.filteredChampions[this.selectedIndex]);
      } else if (this.guess && !this.isCorrect) {
        this.checkGuess();
      }
      this.focusInput();
    },
    handleImageLoad() {
      this.isLoading = false;
    },
    navigateList(direction) {
      if (!this.showSuggestions || !this.filteredChampions.length) return;
      
      this.selectedIndex = (this.selectedIndex + direction + this.filteredChampions.length) % this.filteredChampions.length;
      
      if (this.selectedIndex >= 0) {
        this.guess = this.filteredChampions[this.selectedIndex].name;
      }
    },
    getChampionIconUrl(championId) {
      return this.championIcons[championId] || `https://ddragon.leagueoflegends.com/cdn/${this.version}/img/champion/${championId}.png`;
    },
    focusInput() {
      this.$nextTick(() => {
        if (!this.isCorrect && !this.inputLocked) {
          this.$refs.guessInput.focus();
        }
      });
    },
    handleContainerClick(event) {
      // Only refocus if click wasn't on the input itself
      if (event.target !== this.$refs.guessInput) {
        this.focusInput();
      }
    },
  },
  mounted() {
    this.fetchChampionData();
    this.focusInput();
    
    // Add global click handler
    document.addEventListener('click', (event) => {
      // If click is outside the input and suggestions
      if (!event.target.closest('.autocomplete')) {
        this.showSuggestions = false;
      }
    });
  },
  beforeDestroy() {
    // Clean up global click handler
    document.removeEventListener('click', this.handleGlobalClick);
  },
};
</script>

<style scoped>
.game-container {
  width: 800px; /* Reduced from 1000px */
  min-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.image-container {
  position: relative;
  width: 100%;
  padding-bottom: 75%; /* 4:3 aspect ratio (3/4 = 0.75) */
  overflow: hidden;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  background: #1a1a1a;
  min-width: 760px; /* Reduced from 960px */
  min-height: 570px; /* Reduced to maintain 4:3 ratio (760 * 0.75) */
}

.loading-screen {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: #1a1a1a;
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  z-index: 1;
}

img {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.autocomplete {
  position: relative;
  width: 100%;
  max-width: 300px;
}

.suggestions {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  background: white;
  border: 1px solid #ddd;
  border-top: none;
  max-height: 200px;
  overflow-y: auto;
  z-index: 1000;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  background-color: rgba(0, 0, 0, 0.7);
}

.suggestion-item {
  padding: 8px;
  cursor: pointer;
}

.suggestion-item:hover {
  background-color: white;
  color: black;
}

.champion-wrapper {
  display: flex;
  align-items: center;
  gap: 8px;
  width: 100%;
}

.feedback-message {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  padding: 20px;
  border-radius: 8px;
  font-size: 24px;
  text-align: center;
  z-index: 2;
}

.feedback-message.correct {
  background: rgba(0, 128, 0, 0.8);
  color: white;
}

.feedback-message.incorrect {
  background: rgba(255, 0, 0, 0.8);
  color: white;
}

.champion-icon {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  position: relative;
}

.input-container {
  margin-top: 20px;
  display: flex;
  justify-content: center;
}

input {
  padding: 8px;
  font-size: 16px;
  width: 100%;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.animate-zoom {
  transition: transform 0.5s ease-out;
}

.attempts-counter {
  position: absolute;
  top: 10px;
  right: 10px;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 5px 10px;
  border-radius: 4px;
  font-size: 14px;
}

/* Optional: Add a loading animation */
.loading-screen::after {
  content: '';
  width: 20px;
  height: 20px;
  margin-left: 10px;
  border: 2px solid #ffffff;
  border-radius: 50%;
  border-top-color: transparent;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
</style>
