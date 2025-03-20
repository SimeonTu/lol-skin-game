<template>
  <div class="game-container" @click="handleContainerClick">
    <div class="image-container">
      <!-- Loading screen displayed while the skin image is loading -->
      <div v-if="isLoading" class="loading-screen">
        Loading next champion...
      </div>
      <!-- Champion skin image, shown once loaded -->
      <img v-show="!isLoading" :src="currentSkin" :style="zoomStyle" :class="{ 'animate-zoom': isAnimating }"
        alt="Champion skin" @load="handleImageLoad" />
      <!-- Feedback messages for correct/incorrect guesses -->
      <div v-if="isCorrect" class="feedback-message correct">Correct! It was {{ currentChampion.name }}!</div>
      <div v-if="showIncorrectMessage" class="feedback-message incorrect">Incorrect! Try again...</div>
      <!-- Display the number of attempts made by the user -->
      <div class="attempts-counter">Attempts: {{ attempts }}</div>
    </div>

    <!-- Input container for guessing the champion -->
    <div class="input-container">
      <div class="autocomplete">
        <input ref="guessInput" v-model="guess" @input="filterChampions" @keyup.enter="submitGuess"
          @keydown.down.prevent="navigateList(1)" @keydown.up.prevent="navigateList(-1)"
          placeholder="Guess the champion..." :disabled="isCorrect || inputLocked" />
        <!-- Display filtered champion suggestions -->
        <div v-if="showSuggestions && filteredChampions.length" class="suggestions">
          <div v-for="(champion, index) in filteredChampions" :key="champion.id" @click="selectChampion(champion)"
            :class="['suggestion-item', { 'selected': index === selectedIndex }]">
            <div class="champion-wrapper">
              <!-- Champion icon and name in the suggestion list -->
              <img :src="getChampionIconUrl(champion.id)" :alt="champion.name" class="champion-icon" />
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
      champions: [], // List of all champions
      currentChampion: null, // Currently selected champion
      currentSkin: '', // URL of the current champion's skin
      availableSkins: [], // List of available skins for the current champion
      zoomLevel: 2, // Zoom level for the skin image
      attempts: 0, // Number of attempts made by the user
      guess: '', // User's current guess
      positionX: 0, // X position for the zoomed image
      positionY: 0, // Y position for the zoomed image
      version: '13.24.1', // Game data version
      isCorrect: false, // Whether the user's guess is correct
      showSuggestions: false, // Whether to show the suggestions dropdown
      filteredChampions: [], // Filtered list of champions based on user input
      isAnimating: false, // Whether to animate the zoom effect
      isLoading: true, // Whether the skin image is still loading
      selectedIndex: -1, // Index of the selected suggestion
      showIncorrectMessage: false, // Whether to show the incorrect guess message
      inputLocked: false, // Whether the input is locked (e.g., after an incorrect guess)
      championIcons: {}, // Cache for preloaded champion icons
    };
  },
  computed: {
    // Computed style for the zoomed image
    zoomStyle() {
      return {
        transform: `scale(${this.zoomLevel}) translate(${this.positionX}%, ${this.positionY}%)`,
        transformOrigin: 'center',
      };
    },
  },
  methods: {
    // Fetch champion data from the API
    async fetchChampionData() {
      try {
        const response = await fetch(
          `https://ddragon.leagueoflegends.com/cdn/${this.version}/data/en_US/champion.json`
        );
        const data = await response.json();
        // Map champion data to a simpler format
        this.champions = Object.values(data.data).map(champion => ({
          id: champion.id,
          name: champion.name,
        }));

        // Select a random champion to start the game
        await this.selectRandomChampion();
      } catch (error) {
        console.error('Error fetching champion data:', error);
      }
    },
    // Select a random champion and load their skin
    async selectRandomChampion() {
      this.isLoading = true; // Show loading screen
      const randomChamp = this.champions[Math.floor(Math.random() * this.champions.length)];

      // Fetch detailed champion data including skins
      const response = await fetch(
        `https://ddragon.leagueoflegends.com/cdn/${this.version}/data/en_US/champion/${randomChamp.id}.json`
      );
      const data = await response.json();
      const championData = data.data[randomChamp.id];

      this.currentChampion = randomChamp;
      this.availableSkins = championData.skins;

      // Select a random skin for the champion
      const randomSkin = this.availableSkins[Math.floor(Math.random() * this.availableSkins.length)];
      this.currentSkin = `https://ddragon.leagueoflegends.com/cdn/img/champion/splash/${randomChamp.id}_${randomSkin.num}.jpg`;

      // Reset game state
      this.zoomLevel = 2;
      this.attempts = 0;
      this.isCorrect = false;
      this.guess = '';
      this.isAnimating = false;

      // Randomize the initial zoom position
      const maxOffset = 25;
      this.positionX = Math.random() * maxOffset * 2 - maxOffset;
      this.positionY = Math.random() * maxOffset * 2 - maxOffset;

      // Focus the input field after the next DOM update
      this.$nextTick(() => {
        this.focusInput();
      });
    },
    // Handle image load event
    handleImageLoad() {
      this.isLoading = false; // Hide loading screen
      this.preloadChampionIcons(); // Start preloading champion icons in the background
    },
    // Preload champion icons for the autocomplete suggestions
    preloadChampionIcons() {
      this.champions.forEach(champion => {
        const img = new Image();
        img.src = `https://ddragon.leagueoflegends.com/cdn/${this.version}/img/champion/${champion.id}.png`;
        this.championIcons[champion.id] = img.src;
      });
    },
    // Filter champions based on user input
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
      ).slice(0, 5); // Show only the top 5 matches

      this.showSuggestions = true;
      this.selectedIndex = -1; // Reset selection when filtering
    },
    // Select a champion from the suggestions
    selectChampion(champion) {
      this.guess = champion.name;
      this.showSuggestions = false;
      this.checkGuess(); // Check if the guess is correct
      this.focusInput(); // Refocus the input field
    },
    // Check if the user's guess is correct
    checkGuess() {
      this.attempts++;
      if (this.guess.toLowerCase() === this.currentChampion.name.toLowerCase()) {
        this.isCorrect = true;
        this.isAnimating = true;
        // Reset zoom and position to show the full image
        this.zoomLevel = 1;
        this.positionX = 0;
        this.positionY = 0;

        // Select a new random champion after a delay
        setTimeout(() => {
          this.selectRandomChampion();
        }, 2000);
      } else {
        this.showIncorrectMessage = true;
        this.inputLocked = true;
        this.isAnimating = true;
        // Zoom out slightly on incorrect guess
        this.zoomLevel = Math.max(1, this.zoomLevel - 0.25);
        const zoomProgress = (2 - this.zoomLevel) / 1;
        this.positionX *= (1 - zoomProgress);
        this.positionY *= (1 - zoomProgress);

        // Reset incorrect message and unlock input after a delay
        setTimeout(() => {
          this.showIncorrectMessage = false;
          this.inputLocked = false;
          this.focusInput();
        }, 500);
      }
    },
    // Submit the user's guess
    submitGuess() {
      if (this.selectedIndex >= 0) {
        this.selectChampion(this.filteredChampions[this.selectedIndex]);
      } else if (this.guess && !this.isCorrect) {
        this.checkGuess();
      }
      this.focusInput();
    },
    // Navigate the suggestions list with arrow keys
    navigateList(direction) {
      if (!this.showSuggestions || !this.filteredChampions.length) return;

      this.selectedIndex = (this.selectedIndex + direction + this.filteredChampions.length) % this.filteredChampions.length;

      if (this.selectedIndex >= 0) {
        this.guess = this.filteredChampions[this.selectedIndex].name;
      }
    },
    // Get the URL for a champion's icon
    getChampionIconUrl(championId) {
      return this.championIcons[championId] || `https://ddragon.leagueoflegends.com/cdn/${this.version}/img/champion/${championId}.png`;
    },
    // Focus the input field
    focusInput() {
      this.$nextTick(() => {
        if (!this.isCorrect && !this.inputLocked) {
          this.$refs.guessInput.focus();
        }
      });
    },
    // Handle clicks outside the input field
    handleContainerClick(event) {
      if (event.target !== this.$refs.guessInput) {
        this.focusInput();
      }
    },
  },
  mounted() {
    this.fetchChampionData(); // Fetch champion data when the component is mounted
    this.focusInput(); // Focus the input field initially

    // Add a global click handler to close suggestions when clicking outside
    document.addEventListener('click', (event) => {
      if (!event.target.closest('.autocomplete')) {
        this.showSuggestions = false;
      }
    });
  },
  beforeDestroy() {
    // Clean up the global click handler when the component is destroyed
    document.removeEventListener('click', this.handleGlobalClick);
  },
};
</script>

<style scoped>
.game-container {
  width: 800px;
  /* Reduced from 1000px */
  min-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.image-container {
  position: relative;
  width: 100%;
  padding-bottom: 75%;
  /* 4:3 aspect ratio (3/4 = 0.75) */
  overflow: hidden;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  background: #1a1a1a;
  min-width: 760px;
  /* Reduced from 960px */
  min-height: 570px;
  /* Reduced to maintain 4:3 ratio (760 * 0.75) */
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