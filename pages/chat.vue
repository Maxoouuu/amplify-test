<template>
  <div>
    <!-- Menu Left side -->
    <div class="relative">
      <div class="absolute inset-y-0 left-0 max-w-3xl z-50">
        <Private
          v-for="user in filteredUserList"
          :key="user.id"
          :username="user.id"
          :personal_id="userID"
        />
      </div>
    </div>
    <!-- Chat -->
    <div class="relative">
      <div
        class="flex flex-col items-center justify-center w-screen min-h-screen bg-gray-100 text-gray-800 pt-14 md:pt-16"
      >
        <div
          class="flex flex-col flex-grow w-full max-w-xl bg-white overflow-hidden"
        >
          <div
            class="flex flex-col flex-grow h-0 p-4 overflow-y-auto position-fixed"
            ref="scrollContainer"
          >
            <Message
              v-for="message in messages.slice()"
              :key="message.id"
              :username="message.user_id"
              :receiver_id="message.receiver_id"
              :personal="message.user_id === userID"
              :timestamp="message.timestamp"
              :text="message.text"
              @updateMessages="updateMessages()"
            />
          </div>
          <div class="flex justify-center items-center bg-gray-300 p-4">
            <input
              class="flex items-center h-10 w-full rounded px-3 text-sm"
              type="text"
              placeholder="Type your message…"
              v-model="input"
              @keydown="handleSend"
            />
            <!-- <div>
              <input
                type="file"
                accept="image/png, image/jpeg, image/jpg, image/gif"
                id="fileInput"
                hidden
                @change="onFileChange"
              />
              <label class="custom-upload-btn" for="fileInput">
                <div>
                  <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="32"
                    height="32"
                    viewBox="0 0 512 512"
                  >
                    <path
                      fill="none"
                      stroke="currentColor"
                      stroke-linecap="round"
                      stroke-miterlimit="10"
                      stroke-width="32"
                      d="M216.08 192v143.85a40.08 40.08 0 0 0 80.15 0l.13-188.55a67.94 67.94 0 1 0-135.87 0v189.82a95.51 95.51 0 1 0 191 0V159.74"
                    />
                  </svg>
                </div>
              </label>
            </div>
            <div v-if="selectedFileName">{{ selectedFileName }}</div> -->
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
const user = useUser();
const chat = useChatStore();
const input = ref("");
const userID = ref(null);
const userList = ref([]);
const receiver_id = useReceiver_id();
const messages = ref([]);
const messagesCount = ref(0);
const maxMessagesPerRequest = 50;
const scrollContainer = ref(null);

const getUsers = async () => {
  userList.value = await chat.getUserList();
};

// récupère la liste des utilisateurs
await getUsers();

// Récupère l'id de l'utilisateur courant
userID.value = user.value.id;

// Filtre pour ne pas afficher l'utilisateur courant (sois-même)
const filteredUserList = computed(() => {
  return userList.value.filter((user) => {
    return user.id !== userID.value;
  });
});

// Envoie un message quand on appuie sur entrée
const handleSend = async (event) => {
  if (!event.key || event.key === "Enter") {
    if (input.value) {
      await chat.createNewMessage(userID.value, input.value, receiver_id.value);
      input.value = "";
      // console.log("Receiver id : " + receiver_id.value);
    }
  }
};

// Au changement de la conversation on charge les messages et on écoute les nouveaux messages
watch(receiver_id, async (newReceiver_id) => {
  if (newReceiver_id) {
    // Reset les messages et le compteur de messages chargés pour la nouvelle conversation
    messages.value = [];

    messagesCount.value = 0;

    //function qui Charge les messages
    const loadMessagesBatch = async () => {
      // Charge les messages
      console.log("Receiver id : " + newReceiver_id);
      console.log("User id : " + userID.value);
      const loadedMessages = await chat.getMessagesById(
        newReceiver_id,
        userID.value,
        messagesCount.value,
        maxMessagesPerRequest - 1
      );
      messages.value = [...messages.value, ...loadedMessages];
      messagesCount.value += loadedMessages.length;

      nextTick(() => {
        setTimeout(() => {
          scrollContainer.value.scrollTop = scrollContainer.value.scrollHeight;
        }, 200);
      });
    };
    // appel de la function qui charge les messages
    await loadMessagesBatch();

    /*  console.log(messages.value); */

    // Ecoute les nouveaux messages et les ajoute au début du tableau de messages (pour afficher les messages les plus récents en premier)
    await chat.onNewMessageById(messages, newReceiver_id, userID.value, () => {
      /* messages.value = [newMessage, ...messages.value]; */
      messagesCount.value += 1;
      /* console.log(messages.value) */
      nextTick(() => {
        setTimeout(() => {
          scrollContainer.value.scrollTop = scrollContainer.value.scrollHeight;
        }, 200);
      });
    });
  }
});

// Scroll automatique quand on reçoit un nouveau message
onMounted(() => {
  nextTick(() => {
    setTimeout(() => {
      scrollContainer.value.scrollTop = scrollContainer.value.scrollHeight;
    }, 200);
  });
});

// Update les messages quand on envoie un nouveau message
const updateMessages = async () => {
  await chat.onNewMessageById((newMessage) => {
    messages.value = [newMessage, ...messages.value];
    messagesCount.value += 1;
    nextTick(() => {
      setTimeout(() => {
        scrollContainer.value.scrollTop = scrollContainer.value.scrollHeight;
      }, 200);
    });
  });
};

// Upload de fichier
/* const selectedFileName = ref("");

async function onFileChange(e) {
  const file = e.target.files[0];
  selectedFileName.value = file.name;
 
} */
</script>
