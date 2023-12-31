<script lang="ts" setup>
interface AuthorResponse {
  message: string;
  data: DataResponse<Author>;
}

definePageMeta({
  layout: "admin",
  middleware: ["is-admin"],
});

useHead({
  title: "Admin · Authors",
});

const route = useRoute();
const router = useRouter();
const authorResponse = ref<AuthorResponse>();
const sizePerPage = ref(20);
const currentPage = ref(0);
const selectedIds = ref<number[]>([]);
const showAddAuthorModal = ref(false);
const showEditAuthorModal = ref(false);
const searchInput = ref("");
const authorToEdit = ref<Pick<Author, "id" | "name" | "dob" | "nationality">>({
  id: 0,
  name: "",
  dob: "",
  nationality: "",
});

async function getAuthors() {
  await useApiFetch("/sanctum/csrf-cookie");

  const params = new URLSearchParams({
    page: currentPage.value.toString(),
    sizePerPage: sizePerPage.value.toString(),
    q: searchInput.value,
  });

  const result = await useApiFetch("/api/v1/admin/authors?" + params, {
    headers: {
      Accept: "application/json",
    },
  });

  if (result?.error.value) {
    alert(result.error.value?.data.errors);
    return;
  }
  authorResponse.value = result.data.value as AuthorResponse;
}

async function editAuthor() {
  await useApiFetch("/sanctum/csrf-cookie");

  const result = await useApiFetch(
    `/api/v1/admin/authors/${selectedIds.value[0]}/edit`,
    {
      headers: {
        Accept: "application/json",
      },
    }
  );

  if (result?.error.value) {
    alert(result.error.value?.data.errors);
    return;
  }

  authorToEdit.value.id = selectedIds.value[0];
  authorToEdit.value.name = (result.data.value as any).data.name;
  authorToEdit.value.dob = (result.data.value as any).data.dob;
  authorToEdit.value.nationality = (result.data.value as any).data.nationality;

  showEditAuthorModal.value = true;
}

async function deleteAuthors() {
  let message = "";
  if (selectedIds.value.length === 1) {
    message = `Are you sure you want to delete a author with id ${selectedIds.value[0]}?`;
  } else if (selectedIds.value.length === 2) {
    message = `Are you sure want to delete a authors with id ${selectedIds.value[0]} and ${selectedIds.value[1]}?`;
  } else {
    const lastId = selectedIds.value[selectedIds.value.length - 1];
    const idsString = selectedIds.value
      .slice(0, selectedIds.value.length - 1)
      .join(", ");
    message = `Are you sure you want to delete authors with id ${idsString}, and ${lastId}?`;
  }

  if (confirm(message)) {
    await useApiFetch("/sanctum/csrf-cookie");

    const result = await useApiFetch(
      "/api/v1/admin/authors?ids=" + selectedIds.value.join(","),
      {
        method: "DELETE",
        headers: {
          Accept: "application/json",
        },
      }
    );

    if (result?.error.value) {
      alert(result.error.value?.data.errors);
      return;
    }

    selectedIds.value = [];
    await getAuthors();
  } else {
    selectedIds.value = [];
  }
}

watch([sizePerPage, currentPage], async () => {
  await getAuthors();
});

onMounted(async () => {
  if (route.query.redirectToPage) {
    currentPage.value = parseInt(route.query.redirectToPage.toString());
  } else {
    currentPage.value = 1;
  }

  await getAuthors();
});
</script>

<template>
  <div class="p-4 pt-0">
    <Teleport to="body">
      <ModalsAddAuthor
        v-if="showAddAuthorModal"
        @close="showAddAuthorModal = false"
        @refresh="getAuthors"
      />
      <ModalsEditAuthor
        v-if="showEditAuthorModal"
        :author="authorToEdit"
        @close="showEditAuthorModal = false"
        @refresh="getAuthors"
      />
    </Teleport>

    <div class="w-100">
      <div class="datatable-toolbar d-flex align-items-center mb-3 p-2 gap-2">
        <!-- Search bar -->
        <div class="input-group">
          <input
            type="text"
            class="form-control"
            placeholder="Search"
            aria-label="Search"
            aria-describedby="button-search"
            v-model="searchInput"
            @keyup.enter="getAuthors"
          />
          <button
            class="btn btn-primary"
            type="button"
            id="button-search"
            @click="getAuthors"
          >
            <SearchIcon width="20" height="20" />
          </button>
        </div>

        <!-- Size Per Page -->
        <div class="dropdown">
          <button
            class="btn btn-primary dropdown-toggle"
            type="button"
            id="dropdownMenuButton1"
            data-bs-toggle="dropdown"
            aria-expanded="false"
          >
            Size per page
          </button>
          <ul class="dropdown-menu" aria-labelledby="dropdownMenuButton1">
            <li>
              <button
                class="dropdown-item"
                @click="
                  () => {
                    currentPage = 1;
                    sizePerPage = 20;
                  }
                "
              >
                20
              </button>
            </li>
            <li>
              <button
                class="dropdown-item"
                @click="
                  () => {
                    currentPage = 1;
                    sizePerPage = 50;
                  }
                "
              >
                50
              </button>
            </li>
            <li>
              <button
                class="dropdown-item"
                @click="
                  () => {
                    currentPage = 1;
                    sizePerPage = 100;
                  }
                "
              >
                100
              </button>
            </li>
          </ul>
        </div>
      </div>

      <!-- Data Table -->
      <div class="card">
        <h5
          class="card-header d-flex align-items-center justify-content-between"
        >
          <div class="fs-5">Authors</div>
          <div class="d-flex align-items-center gap-2">
            <button
              class="btn btn-primary btn-sm"
              @click="showAddAuthorModal = true"
            >
              <PlusIcon />
            </button>
            <button
              class="btn btn-success btn-sm"
              :disabled="!selectedIds.length"
              @click="editAuthor"
            >
              <PencilIcon />
            </button>
            <button
              class="btn btn-danger btn-sm"
              :disabled="!selectedIds.length"
              @click="deleteAuthors"
            >
              <TrashIcon />
              ({{ selectedIds.length }})
            </button>
            <button
              class="btn btn-light btn-sm d-flex align-items-center gap-2"
              @click="selectedIds = []"
              :disabled="!selectedIds.length"
            >
              <XIcon />
              Unselect all
            </button>
          </div>
        </h5>
        <div
          class="table-responsive text-nowrap"
          style="max-height: calc(100vh - 300px)"
        >
          <table class="table table-striped" v-if="authorResponse">
            <thead>
              <tr>
                <th>
                  <div class="form-check">
                    <input
                      class="form-check-input"
                      type="checkbox"
                      @change="(e) => {
                        if((e.target as HTMLInputElement).checked && authorResponse) {
                          selectedIds = authorResponse?.data.data.map(item => item.id);
                        } else selectedIds = [];
                      }"
                      :checked="
                        selectedIds.length === authorResponse.data.data.length
                      "
                      style="transform: scale(1.4)"
                    />
                  </div>
                </th>
                <th>ID</th>
                <th>Name</th>
                <th>Date of Birth</th>
                <th>Nationality</th>
              </tr>
            </thead>
            <tbody class="table-border-bottom-0">
              <tr v-for="book in authorResponse?.data.data">
                <td>
                  <div class="form-check">
                    <input
                      class="form-check-input"
                      type="checkbox"
                      @change="(e) => {
                        if((e.target as HTMLInputElement).checked) {
                          selectedIds.push(book.id);
                        } else selectedIds = selectedIds.filter(item => item !== book.id);
                      }"
                      :checked="selectedIds.includes(book.id)"
                      :id="`mark-${book.id}`"
                      style="transform: scale(1.4)"
                    />
                  </div>
                </td>
                <td>{{ book.id }}</td>
                <td>{{ book.name }}</td>
                <td>{{ book.dob }}</td>
                <td>{{ book.nationality }}</td>
              </tr>
            </tbody>
          </table>

          <!-- Table Loading placeholder-->
          <div
            :style="{
              height: '100vh',
              width: '100%',
              display: 'grid',
              placeItems: 'center',
            }"
            v-else
          >
            <div class="spinner-border" role="status">
              <span class="visually-hidden">Loading...</span>
            </div>
          </div>
        </div>
      </div>

      <!-- Footer -->
      <div class="d-flex justify-content-between mt-2">
        <div v-if="authorResponse">
          Showing
          {{
            authorResponse?.data.per_page! * authorResponse?.data.current_page!
          }}/{{ authorResponse?.data.total }} entries
        </div>
        <div v-else>Loading...</div>

        <!-- Pagination nav -->
        <nav>
          <ul class="pagination pagination-sm">
            <li class="page-item" v-for="page in authorResponse?.data.links">
              <div
                :class="[
                  'page-link',
                  page.active ? 'active' : '',
                  page.url ? '' : 'disabled',
                ]"
                style="cursor: pointer"
                @click="
                  currentPage = page.label.includes('Prev')
                    ? authorResponse?.data.current_page! - 1
                    : page.label.includes('Next')
                    ? authorResponse?.data.current_page! + 1
                    : parseInt(page.label)
                "
                :title="`page ${page.label.includes('Prev') ? authorResponse?.data.current_page! - 1 : page.label.includes('Next') ? authorResponse?.data.current_page! + 1 : parseInt(page.label)}`"
              >
                {{
                  page.label
                    .replaceAll("&amp;laquo;", "")
                    .replaceAll("&amp;raquo;", "")
                }}
              </div>
            </li>
          </ul>
        </nav>
      </div>
    </div>
  </div>
</template>
