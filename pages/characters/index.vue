<template>
  <v-container fluid :class="darkMode ? 'theme--dark' : ''">
    <!-- Encabezado: búsqueda, filtrado y modo oscuro -->
    <v-row class="my-4" align="center">
      <v-col cols="12" md="4">
        <v-text-field
          v-model="searchQuery"
          label="Buscar por nombre"
          clearable
          outlined
          prepend-icon="mdi-magnify"
        />
      </v-col>
      <v-col cols="12" md="3">
        <v-select
          v-model="filterStatus"
          :items="statusOptions"
          label="Filtrar por estado"
          clearable
          outlined
          prepend-icon="mdi-alert-circle"
        />
      </v-col>
      <v-col cols="12" md="3">
        <v-select
          v-model="filterGender"
          :items="genderOptions"
          label="Filtrar por género"
          clearable
          outlined
          prepend-icon="mdi-gender-male-female"
        />
      </v-col>
      <v-col cols="12" md="2" class="text-right">
        <v-switch v-model="darkMode" label="Modo Oscuro" />
      </v-col>
    </v-row>

    <!-- Botón para abrir el modal de agregar personaje -->
    <v-row class="my-4">
      <v-col cols="12" class="text-right">
        <v-btn color="primary" @click="openCreateModal">
          <v-icon left>mdi-plus</v-icon>Agregar Personaje
        </v-btn>
      </v-col>
    </v-row>

    <!-- Modal para el formulario (crear o editar) -->
    <v-dialog v-model="dialog" max-width="600">
      <v-card>
        <v-card-title class="headline">
          <v-icon left v-if="modalMode === 'create'">mdi-plus</v-icon>
          <v-icon left v-else>mdi-pencil</v-icon>
          {{
            modalMode === "create" ? "Agregar Personaje" : "Editar Personaje"
          }}
        </v-card-title>
        <v-card-text>
          <CharacterForm
            :mode="modalMode"
            :initialData="selectedCharacter"
            @save="handleSave"
            @cancel="dialog = false"
          />
        </v-card-text>
      </v-card>
    </v-dialog>

    <!-- Diálogo de confirmación para editar personaje de la API -->
    <v-dialog v-model="confirmDialog" max-width="400">
      <v-card>
        <v-card-title class="headline">
          <v-icon left>mdi-pencil</v-icon>Confirmar Edición
        </v-card-title>
        <v-card-text>
          Al editar un personaje de la API se creará una copia local con los
          cambios. ¿Desea continuar?
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="primary" @click="confirmEdit">
            <v-icon left>mdi-check</v-icon>Sí, continuar
          </v-btn>
          <v-btn variant="text" @click="cancelConfirmEdit">
            <v-icon left>mdi-cancel</v-icon>Cancelar
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <!-- Lista de tarjetas -->
    <v-row>
      <v-col
        v-for="char in filteredCharacters"
        :key="char.id"
        cols="12"
        sm="6"
        md="4"
        lg="3"
      >
        <v-card max-width="344" class="mx-auto my-4">
          <v-img
            :src="char.image"
            height="200px"
            class="white--text align-end"
            gradient="to bottom, rgba(0,0,0,.1), rgba(0,0,0,.5)"
          ></v-img>

          <!-- Título y subtítulo -->
          <v-card-title>
            <v-icon left>mdi-account</v-icon>{{ char.name }}
          </v-card-title>
          <v-card-subtitle v-if="char.isLocal">
            <v-icon left small>mdi-note-text</v-icon>
            {{ char.description || "Personaje Local" }}
          </v-card-subtitle>
          <v-card-subtitle v-else>
            <v-icon left small>mdi-television-classic</v-icon>
            Rick and Morty
          </v-card-subtitle>

          <!-- Acciones principales -->
          <v-card-actions>
            <v-btn color="error" @click="deleteCharacter(char.id)">
              <v-icon left>mdi-delete</v-icon>Eliminar
            </v-btn>
            <v-btn color="secondary" @click="openEditModal(char)">
              <v-icon left>mdi-pencil</v-icon>Editar
            </v-btn>
            <v-spacer></v-spacer>
            <!-- Botón para expandir o contraer el desglose -->
            <v-btn icon @click="toggleExpand(char.id)">
              <v-icon>
                {{
                  expandedCards.has(char.id)
                    ? "mdi-chevron-up"
                    : "mdi-chevron-down"
                }}
              </v-icon>
            </v-btn>
          </v-card-actions>

          <!-- Desglose expandible: detalles adicionales -->
          <v-slide-y-transition>
            <div v-if="expandedCards.has(char.id)" class="card-details">
              <div class="detail-item">
                <strong>Género:</strong> {{ char.gender || "Desconocido" }}
              </div>
              <div class="detail-item">
                <strong>Especie:</strong> {{ char.species || "Desconocido" }}
              </div>
              <div class="detail-item">
                <strong>Estado:</strong> {{ char.status || "Desconocido" }}
              </div>
              <div class="detail-item">
                <strong>Origen:</strong> {{ char.origin || "Desconocido" }}
              </div>
              <div class="detail-item">
                <strong>Ubicación:</strong> {{ char.location || "Desconocido" }}
              </div>
            </div>
          </v-slide-y-transition>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script setup lang="ts">
import { ref, computed, watch } from "vue";
import { useTheme } from "vuetify";
import { useCharacters } from "@/composables/useCharacters";
import CharacterForm from "@/components/CharacterForm.vue";

interface CharacterFormData {
  id?: number;
  name: string;
  description: string;
  image: string;
  status?: string;
  species?: string;
  gender?: string;
  origin?: string;
  location?: string;
}

function transformApiCharacter(character: any): CharacterFormData {
  return {
    id: character.id,
    name: character.name,
    description: "", // La API no tiene campo "description"
    image: character.image,
    status: character.status,
    species: character.species,
    gender: character.gender,
    origin: character.origin?.name || "",
    location: character.location?.name || "",
  };
}

const dialog = ref(false);
const modalMode = ref<"create" | "edit">("create");
const selectedCharacter = ref<
  Partial<CharacterFormData & { id: number }> | undefined
>(undefined);
const confirmDialog = ref(false);

const { allCharacters, createCharacter, updateCharacter, deleteCharacter } =
  useCharacters();

const pendingEditCharacter = ref<any>(null);
const expandedCards = ref<Set<number>>(new Set());

// Búsqueda y filtros
const searchQuery = ref("");
const filterStatus = ref<string | null>(null);
const filterGender = ref<string | null>(null);
const statusOptions = ["Alive", "Dead", "unknown"];
const genderOptions = ["Female", "Male", "Genderless", "unknown"];

const filteredCharacters = computed(() => {
  return allCharacters.value.filter((char) => {
    const matchesQuery = char.name
      .toLowerCase()
      .includes(searchQuery.value.toLowerCase());
    const matchesStatus = filterStatus.value
      ? char.status === filterStatus.value
      : true;
    const matchesGender = filterGender.value
      ? char.gender === filterGender.value
      : true;
    return matchesQuery && matchesStatus && matchesGender;
  });
});

// Modo oscuro usando useTheme de Vuetify
const darkMode = ref(false);
const theme = useTheme();
watch(darkMode, (val) => {
  theme.global.name.value = val ? "dark" : "light";
});

function toggleExpand(id: number) {
  if (expandedCards.value.has(id)) {
    expandedCards.value.delete(id);
  } else {
    expandedCards.value.add(id);
  }
}

function openCreateModal() {
  modalMode.value = "create";
  selectedCharacter.value = undefined;
  dialog.value = true;
}

function openEditModal(character: any) {
  if (!character.isLocal) {
    pendingEditCharacter.value = transformApiCharacter(character);
    confirmDialog.value = true;
  } else {
    modalMode.value = "edit";
    selectedCharacter.value = { ...character };
    dialog.value = true;
  }
}

function confirmEdit() {
  if (pendingEditCharacter.value) {
    modalMode.value = "edit";
    selectedCharacter.value = { ...pendingEditCharacter.value };
    dialog.value = true;
  }
  confirmDialog.value = false;
  pendingEditCharacter.value = null;
}

function cancelConfirmEdit() {
  confirmDialog.value = false;
  pendingEditCharacter.value = null;
}

function handleSave(data: CharacterFormData) {
  if (modalMode.value === "create") {
    createCharacter(data);
  } else {
    if (selectedCharacter.value?.id !== undefined) {
      updateCharacter(selectedCharacter.value.id, data);
    }
  }
  dialog.value = false;
}
</script>

<style scoped>
/* Transición para el contenido expandible */
.v-slide-y-transition-enter-active,
.v-slide-y-transition-leave-active {
  transition: all 0.15s ease-in-out;
}

/* Estilos para el contenido expandible en modo claro */
.card-details {
  font-size: 0.9rem;
  overflow: hidden;
  padding: 0.5rem 1rem;
  background-color: #f9f9f9;
  border-top: 1px solid #e0e0e0;
}

/* Estilos para el contenido expandible en modo oscuro */
.theme--dark .card-details {
  background-color: #424242;
  color: #ffffff;
  border-top: 1px solid #616161;
}

.detail-item {
  margin: 0.25rem 0;
}
</style>
