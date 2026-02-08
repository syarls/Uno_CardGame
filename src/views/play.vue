<template>
  <div class="background">
    <div class="playerturn">Player {{ currentPlayerIndex + 1 }}'s Turn</div>
    <div id="uno-game">
      <div class="container">
        <div class="player-grid">
          <div class="player-hand" v-for="(player, index) in players" :key="index">
            <div class="player-info">
              <div class="playercontainer">
                <div class="numberOfCards">{{ player.hand.length }}</div>
              </div>
              <h2 class="players">Player {{ index + 1 }}:</h2>
            </div>

            <div v-if="index === myIndex">
              <div class="card" v-for="(card, i) in player.hand.slice(startIndex, startIndex + cardsPerPage)" :key="i"
                @click="index === currentPlayerIndex && playCard(i + startIndex)">
                <img :src="getCardImage(card)" class="card-image" />
              </div>
            </div>

            <div v-else class="card-back-grid">
              <div class="card" v-for="n in player.hand.length" :key="n">
                <img src="/src/assets/images/back.png" class="card-image" />
              </div>
            </div>
          </div>
        </div>
      </div>
      <div style="color: red; font-size: 12px">//
        index: {{ index }},
        myIndex: {{ myIndex }},
        currentPlayerIndex: {{ currentPlayerIndex }}
      </div>//
      <div class="current-card">
        <h2>Last Card</h2>
        <div class="card-current">
          <nobr>
            <img :src="getCardImage(currentCard)" :alt="getCardAltText(currentCard)" class="card-image" />
            <img src="../assets/images/unoDeck.png" class="deck" @click="drawCard" />
          </nobr>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref } from "vue";
import sweetNotif from "sweetalert2";
import randomCatGif from "../assets/images/random-cat.gif";
import congratulations from "../assets/images/random-cat4.gif";
import bgmusic from "../assets/audio/main-bg.mp3";
import io from "socket.io-client";
import axios from "axios";

export default {
  data() {
    return {
      colors: [],
      values: [],
      currentPlayerIndex: 0,
      players: [],
      currentCard: { color: "", value: "" },
      direction: "normal",
      myIndex: null,
      selectedColor: null,
      backgroundMusic: null,
      socket: null,
      cardsPerPage: 5,
      startIndex: 0,
    };
  },

  mounted() {

    this.setupWebSocket()
    this.myIndex = Number(this.$route.query.myIndex);

    axios
      .get("http://localhost:5174/current-card")
      .then((response) => {
        this.currentCard = response.data;
      })
      .catch((error) => {
        console.error("Error fetching current card:", error);
      });

    axios
      .get("http://localhost:5174/players")
      .then((response) => {
        this.players = response.data;
      })
      .catch((error) => {
        console.error("Error fetching players data:", error);
      });

    axios
      .get("http://localhost:5174/colors")
      .then((response) => {
        this.colors = response.data.colors;
      })
      .catch((error) => {
        console.error("Error fetching colors data:", error);
      });

    axios
      .get("http://localhost:5174/values")
      .then((response) => {
        this.values = response.data.values;
      })
      .catch((error) => {
        console.error("Error fetching values data:", error);
      });

    this.backgroundMusic = new Audio(bgmusic);
    this.backgroundMusic.loop = true;
    this.backgroundMusic.volume = 0.1;
    this.backgroundMusic.play();
  },

  beforeUnmount() {
    this.backgroundMusic.pause();
  },

  methods: {
    playCard(cardIndex) {
      console.log("Are you being triggered", "yes")
      this.socket.emit("play-card", { cardIndex });
    },

    drawCard() {
      this.socket.emit("draw-card");
    },

    getCardImage(card) {
      const imageUrl = `/src/assets/images/${card.color}_${card.value}.png`;
      return imageUrl;
    },

    getCardAltText(card) {
      return `${card.color} ${card.value}`;
    },

    previousCards() {
      const currentPlayerHand = this.players[this.currentPlayerIndex];

      if (this.startIndex >= 0) {
        this.startIndex -= this.cardsPerPage;
        if (this.startIndex < 0) {
          this.startIndex = 0;
        }
      }
    },

    nextCards() {
      const currentPlayerHand = this.players[this.currentPlayerIndex];
      const maxIndex = Math.max(currentPlayerHand.length - this.cardsPerPage, 0);
      if (this.startIndex + this.cardsPerPage < currentPlayerHand.length) {
        this.startIndex += this.cardsPerPage;
        if (this.startIndex > maxIndex) {
          this.startIndex = maxIndex;
        }
      }
    },
    setupWebSocket(){
       const direction = ref("normal");

    let currentCard = { color: "", value: "" };

    this.socket = io("ws://localhost:5174");

    this.socket.on("players", (data) => {
      console.log("Players", data)
      this.players = data;
    });

    this.socket.on("my-index", (index) => {
      console.log("index", index)
      this.myIndex = index;
    });

    this.socket.on("currentPlayerIndex", (index) => {
      console.log("Received currentPlayerIndex:", index);
      currentPlayerIndex.value = index;
    });

    this.socket.on("currentCard", (card) => {
      console.log("Received currentCard:", card);
      currentCard = card;
    });

    this.socket.on("direction", (dir) => {
      console.log("Received direction:", dir);
      direction.value = dir;
    });

    this.socket.on("game-state", (state) => {
      this.players = state.players;
      this.currentPlayerIndex = state.currentPlayerIndex;
      this.currentCard = state.currentCard;
      this.direction = state.direction;
    });

    this.socket.on("player-updated", (updatedPlayers) => {
      console.log("Received updated player data:", updatedPlayers);

      this.players = updatedPlayers;
    });

    this.socket.on("switchPlayer", (playerIndex) => {
      if (this.direction === "normal") {
        this.currentPlayerIndex = (playerIndex + 1) % this.players.length;
      } else if (this.direction === "reverse") {
        this.currentPlayerIndex =
          (playerIndex - 1 + this.players.length) % this.players.length;
      }
    });

    this.socket.on("invalid-move", () => {
      sweetNotif.fire({
        title: "Invalid move!",
        text: "You can only play a card of the same color or value.",
        imageUrl: `${randomCatGif}`,
        imageWidth: 300,
        imageHeight: 250,
      });
      return;
    });

    this.socket.on("game-ended", () => {
      sweetNotif.fire({
          title: "Congratulations!",
          text: `Player ${this.currentPlayerIndex + 1} won the game!`,
          imageUrl: `${congratulations}`,
          imageWidth: 300,
          imageHeight: 300,
          timer: 30000,
          timerProgressBar: true,
        })
        .then((result) => {
          if (result.dismiss === sweetNotif.DismissReason.cancel || sweetNotif.DismissReason.timer) {
            this.socket.disconnect()
            this.$router.push("/main");
          }
        });
      return;
    });
    }
  },
};

</script>

<style scoped>
#uno-game {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.container {
  margin-top: 50px;
}

.player-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  grid-template-rows: repeat(2, 1fr);
  row-gap: 50px;
  column-gap: 170px;
}

h2 {
  color: white;
}

.player-hand {
  margin-bottom: 20px;
}

.card,
.card-current {
  width: 130px;
  height: 190px;
  border-radius: 5px;
  margin: -15px;
  display: inline-block;
  position: relative;
  cursor: pointer;
}

.current-card .card {
  width: 120px;
  height: 180px;
  margin-bottom: 20px;
}

.playerturn {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: auto;
  font-size: 24px;
  font-weight: bold;
  color: white;
  z-index: 100;
  background-image: linear-gradient(1deg, #4f58fd, #149bf3 99%);
}

.card:hover {
  transform: translateY(-15px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0);
}

.player-hand {
  margin: 0 10px;
}

.card-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.card-image:hover {
  transform: scale(1.05);
}

.background {
  background-image: url("../assets/background/backgroundPlay.gif");
  position: relative;
  min-height: 100vh;
  min-width: 100vw;
  background-size: cover;
  background-position: center;
}

.deck {
  width: 200px;
  height: 200px;
}

.deck:hover {
  transform: scale(1.05);
}

.player-info {
  display: flex;
  align-items: center;
  margin-bottom: 20px;
}

.numberOfCards {
  color: #f8db23;
  padding: 0.25em 0.5em;
  margin-left: 1em;
  margin-top: 30px;
  font-size: 26px;
  font-weight: bold;
  border-radius: 30px;
  height: 15px;
  width: 5px;
  text-align: center;
  line-height: 20px;
  background-color: red;
  text-indent: -0.15em;
  text-shadow: -1px -1px 0 #000, 1px -1px 0 #000, -1px 1px 0 #000, 1px 1px 0 #000;
}

.playercontainer {
  position: relative;
  background-image: url("../assets/characters/default-avatar.png");
  background-size: cover;
  height: 50px;
  width: 50px;
  border-radius: 5px;
}

.players {
  margin-left: 15px;
  font-family: Cambria, Cochin, Georgia, Times, "Times New Roman", serif;
}

.prev-button {
  position: absolute;
  top: 40%;
  left: 30px;
  transform: translateY(-50%);
}

.next-button {
  position: absolute;
  top: 40%;
  right: 30px;
  transform: translateY(-50%);
}

.next-button,
.prev-button {
  align-items: center;
  background-image: linear-gradient(1deg, #4f58fd, #149bf3 99%);
  border: 0;
  border-radius: 10px;
  box-sizing: border-box;
  color: #fff;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  font-family: "Codec cold", sans-serif;
  font-size: 16px;
  font-weight: 700;
  height: 54px;
  justify-content: center;
  letter-spacing: 0.4px;
  line-height: 1;
  max-width: 100%;
  padding-left: 20px;
  padding-right: 20px;
  padding-top: 3px;
  text-decoration: none;
  text-transform: uppercase;
  user-select: none;
  -webkit-user-select: none;
  touch-action: manipulation;
}
</style>
