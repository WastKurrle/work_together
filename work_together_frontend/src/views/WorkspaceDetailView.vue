<template>
  <div class="mt-3 container">
    <h4 class="text-center">{{ workspace.name }}</h4>
    <p class="text-center"><b>Description:</b> {{ workspace.description }}</p>

    <div class="border p-3 mb-3">
        <div class="d-flex flex-column">
            <button class="btn btn-success mb-3" data-bs-target="#chatModal" data-bs-toggle="modal" @click="startCallChat">Show Chat</button>
            <button class="btn btn-success mb-3" data-bs-target="#uploadFileModal" data-bs-toggle="modal">Upload File</button>
            <button class="btn btn-warning" @click="$router.push(`/organization/${$route.params.org_slug}`)">Back</button>
        </div>
    </div>

    <div class="modal fade" id="uploadFileModal" aria-hidden="true" aria-labelledby="exampleModalToggleLabel2">
        <div class="modal-dialog modal-dialog-centered">
          <div class="modal-content">
            <div class="modal-header">
              <h5 class="modal-title" id="createWorkspaceModal">Upload File</h5>
              <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
            </div>
            <div class="modal-body">
                <form enctype="multipart/form-data" @submit.prevent="uploadFile" method="POST">
                    <input type="file" ref="file" @change="selectFile" class="form-control mb-3">

                    <label for="desc" class="form-label">Description</label>
                    <textarea id="desc" cols="30" rows="5" class="form-control mb-3" v-model="file_desc"></textarea>

                    <div class="bg-danger text-white p-3 mb-3 rounded" v-if="upload_errors.length">
                        <p v-for="error in upload_errors" :key="error">{{ error }}</p>
                    </div>

                    <button class="btn btn-warning" :disabled="file == ''">Upload</button>
                </form>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
            </div>
          </div>
        </div>
    </div>

    <div class="modal fade" id="chatModal" aria-hidden="true" aria-labelledby="exampleModalToggleLabel2">
        <div class="modal-dialog modal-dialog-centered modal-dialog-scrollable">
          <div class="modal-content">
            <div class="modal-header">
              <h5 class="modal-title" id="createWorkspaceModal">Chat from {{ workspace.name }}</h5>
              <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close" @click="destroyCallChat"></button>
            </div>
            <div class="modal-body">
                <div class="mb-3" id="chat">
                    <ChatMessage v-for="message in chatMessages" :key="message.id" :messageObj="message"/>
                </div>

                <form method="POST" @submit.prevent="postMessage">
                    <input type="text" class="form-control mb-3" placeholder="message" v-model="message">

                    <div
                        class="bg-danger text-white p-3 mb-3 mt-3 rounded"
                        v-if="postMessageErrors.length"
                        >
                        <p v-for="error in postMessageErrors" :key="error">{{ error }}</p>
                    </div>

                    <button class="btn btn-primary">Send</button>
                </form>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal" @click="destroyCallChat">Close</button>
            </div>
          </div>
        </div>
    </div>

    <div class="border p-3 mb-3">
        <h5 class="text-center">File uploads</h5>

        <div class="card mb-3" v-for="uploadedFile in uploadedFiles" :key="uploadedFile.id">
            <div class="card-body">
              <h5 class="card-title">{{ uploadedFile.get_file_name }}</h5>
              <p>{{ uploadedFile.description }}</p>
              <button class="btn btn-primary" @click="downloadFile(uploadedFile.get_file, uploadedFile.get_file_name)">Download</button>
            </div>
        </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import Toastify from "toastify-js";
import ChatMessage from '../components/ChatMessage.vue'

export default {
    name: 'WorkspaceDetailView',

    data() {
        return {
            workspace: {},
            uploadedFiles: [],
            file: '',
            upload_errors: [],
            file_desc: '',

            // chat
            message: '',
            chatMessages: [],
            postMessageErrors: [],
            chatCallTimer: 0
        }
    },

    components: {
        ChatMessage
    }, 

    methods: {
        async getWorkspace() {
            const org_slug = this.$route.params.org_slug
            const workspace_slug = this.$route.params.workspace_slug

            axios
                .get(`/api/workspace/get/${org_slug}/${workspace_slug}/`)
                .then(response => {
                    this.workspace = response.data
                    this.getUploadedFiles()
                    this.getChatMessages()
                })
                .catch(error => {
                    if (error.response.status === 403) {
                        Toastify({
                            text: "You don`t have permissons",
                            duration: 3000,
                            close: true,
                            gravity: "bottom",
                            position: "right",
                            stopOnFocus: true,

                            style: {
                                background: "red",
                            },
                        }).showToast();
                    }
                })
        },

        selectFile(event) {
            if (event.target.files.length > 0) {
                this.file = event.target.files[0]
            } else {
                this.file = ''
            }
        },

        async uploadFile() {
            this.upload_errors = []

            if (this.file_desc.trim() == '') {
                this.upload_errors.push('The description field is empty')
            } else {
                const formData = {
                    file: this.file,
                    description: this.file_desc
                }

                await axios
                    .post(`/api/workspace/${this.workspace.id}/file-upload/`, formData, {
                        headers: {
                            "Content-Type": "multipart/form-data",
                        }
                    })
                    .then(response => {
                        console.log(response)
                        Toastify({
                            text: "File has been uploaded",
                            duration: 3000,
                            close: true,
                            gravity: "bottom",
                            position: "right",
                            stopOnFocus: true,

                            style: {
                                background: "green",
                            },
                        }).showToast();

                        this.file = '',
                        this.file_desc = ''
                    })
                    .catch(error => {
                        console.log(error)
                        this.upload_errors.push('Something went wrong')
                    })
            }
        },

        async getUploadedFiles() {
            axios
                .get(`/api/workspace/${this.workspace.id}/file-get/`)
                .then(response => {
                    this.uploadedFiles = response.data
                })
                .catch(error => {
                    console.log(error)
                })
        },

        async downloadFile(url, fileName) {
            axios
                .get(url, {
                    responseType: 'blob'
                })
                .then(response => {
                    const fileUrl = window.URL.createObjectURL(new Blob([response.data]))
                    const fileLink = document.createElement('a')

                    fileLink.href = fileUrl
                    fileLink.setAttribute('download', fileName)
                    document.body.appendChild(fileLink)

                    fileLink.click()
                })
        },

        async postMessage() {
            this.postMessageErrors = []

            if (this.message.trim() === '') {
                this.postMessageErrors.push('Message field is empty')
                return
            }

            const formData = {
                message: this.message,
                user: parseInt(localStorage.getItem('userId')),
                workspace: this.workspace.id
            }

            axios.post('/api/workspace/messages/post/', formData)
                .then(response => {
                    console.log(response.data);
                    this.message = ''
                })
                .catch(error => {
                    console.log(error)
                    this.postMessageErrors.push(error.response.data)
                })
        },

        async getChatMessages() {
            axios.get(`/api/workspace/${this.workspace.id}/messages/`)
                .then(response => {
                    this.chatMessages = response.data
                })
                .catch(error => {
                    console.log(error)
                })
        },

        startCallChat() {
            this.chatCallTimer = setInterval(this.getChatMessages, 1000)
        },

        destroyCallChat() {
            clearInterval(this.chatCallTimer)
        }
    },

    mounted() {
        this.getWorkspace()
    },
}
</script>

<style>

</style>