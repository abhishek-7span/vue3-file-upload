<template>
  <div class="usage-example">
    <h1>File Uploader Example</h1>

    <FileUploader
      :multiple="true"
      :maxFileSize="5 * 1024 * 1024"
      :maxTotalSize="20 * 1024 * 1024"
      :maxFiles="5"
      :allowedTypes="['.jpg', '.png', '.pdf']"
      :allowedMimeTypes="['image/jpeg', 'image/png', 'application/pdf']"
      uploadUrl="/api/upload"
      @fileSelected="handleFileSelected"
      @fileRemoved="handleFileRemoved"
      @uploadProgress="handleUploadProgress"
      @uploadSuccess="handleUploadSuccess"
      @uploadError="handleUploadError"
      @validationError="handleValidationError"
      uploaderClass="my-custom-uploader-class"
      previewClass="my-custom-preview-class"
      itemClass="my-custom-item-class"
      errorClass="my-custom-error-class"
      :validateFile="customValidationExample"
      ref="fileUploaderRef"
    />

    <div v-if="allSelectedFiles.length > 0" class="selected-files-list">
      <h2>Selected Files:</h2>
      <ul>
        <li v-for="(file, index) in allSelectedFiles" :key="file.name + index">
          {{ file.name }} ({{ (file.size / 1024).toFixed(2) }} KB) - Status:
          {{ file.status }}
          <span v-if="file.status === 'uploading'">
            ({{ file.progress.toFixed(0) }}%)</span
          >
        </li>
      </ul>
      <button @click="uploadAllFiles" :disabled="isUploading">
        {{ isUploading ? "Uploading..." : "Upload All Files" }}
      </button>
    </div>

    <div v-if="componentErrors.length > 0" class="component-errors">
      <h2>Component Errors:</h2>
      <ul>
        <li v-for="(error, index) in componentErrors" :key="index">
          {{ error }}
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import { ref, computed } from "vue";
import FileUploader from "../vue-file-upload/FileUploader.vue"; // Adjust the path as needed

export default {
  name: "UsageExample",
  components: {
    FileUploader,
  },
  setup() {
    const allSelectedFiles = ref([]); // To store files received from fileSelected event
    const componentErrors = ref([]); // To store errors received from validationError event
    const isUploading = ref(false);

    const handleFileSelected = (files) => {
      console.log("Files selected:", files);
      // Update the list of selected files.
      // Note: FileUploader manages its internal selectedFiles state.
      // This list here is just for demonstration in the parent component.
      // You might want to merge or replace based on your needs and the 'multiple' prop.
      allSelectedFiles.value = files; // Assuming fileSelected emits the current list
    };

    const handleFileRemoved = (file) => {
      console.log("File removed:", file);
      // Update the list in the parent component
      allSelectedFiles.value = allSelectedFiles.value.filter((f) => f !== file);
    };

    const handleUploadProgress = ({ file, progress }) => {
      console.log(`Upload progress for ${file.name}: ${progress}%`);
      // Find the file in allSelectedFiles and update its progress
      const fileIndex = allSelectedFiles.value.findIndex((f) => f === file);
      if (fileIndex !== -1) {
        allSelectedFiles.value[fileIndex].progress = progress;
      }
    };

    const handleUploadSuccess = ({ file, response }) => {
      console.log(`Upload success for ${file.name}:`, response);
      // Find the file in allSelectedFiles and update its status
      const fileIndex = allSelectedFiles.value.findIndex((f) => f === file);
      if (fileIndex !== -1) {
        allSelectedFiles.value[fileIndex].status = "success";
      }
    };

    const handleUploadError = ({ file, error, status, response }) => {
      console.error(`Upload error for ${file.name}:`, error, status, response);
      // Find the file in allSelectedFiles and update its status
      const fileIndex = allSelectedFiles.value.findIndex((f) => f === file);
      if (fileIndex !== -1) {
        allSelectedFiles.value[fileIndex].status = "error";
      }
    };

    const handleValidationError = (errors) => {
      console.error("Validation errors:", errors);
      componentErrors.value = errors; // Display validation errors from the component
    };

    const uploadAllFiles = async () => {
      isUploading.value = true;
      // Filter files that are pending or had an error
      const filesToUpload = allSelectedFiles.value.filter(
        (file) => file.status === "pending" || file.status === "error"
      );

      for (let i = 0; i < filesToUpload.length; i++) {
        const file = filesToUpload[i];
        const fileIndexInSelected = allSelectedFiles.value.findIndex(
          (f) => f === file
        );
        if (fileIndexInSelected !== -1) {
          // Call the exposed uploadFile method on the FileUploader instance
          if (fileUploaderRef.value && fileUploaderRef.value.uploadFile) {
            fileUploaderRef.value.uploadFile(file);
          } else {
            console.error(
              "FileUploader component or uploadFile method not available."
            );
          }
        }
      }
      // isUploading.value = false; // Set to false when all uploads are initiated (not necessarily finished)
    };

    // Let's add a simple custom validation example
    const customValidationExample = (file) => {
      // Example: Reject files with 'badword' in the name
      if (file.name.toLowerCase().includes("badword")) {
        return false; // Validation fails
      }
      // Example: Async validation (e.g., check file content on server)
      // return fetch('/api/validate-file', { method: 'POST', body: file })
      //     .then(response => response.json())
      //     .then(data => data.isValid);
      return true; // Validation passes
    };

    return {
      allSelectedFiles,
      componentErrors,
      isUploading,
      handleFileSelected,
      handleFileRemoved,
      handleUploadProgress,
      handleUploadSuccess,
      handleUploadError,
      handleValidationError,
      // uploadAllFiles, // Removed for simplicity in this example
      customValidationExample, // Expose custom validation function
    };
  },
};
</script>

<style scoped>
.usage-example {
  font-family: sans-serif;
  padding: 20px;
}

h1,
h2 {
  color: #333;
}

.selected-files-list {
  margin-top: 30px;
  border-top: 1px solid #eee;
  padding-top: 20px;
}

.selected-files-list ul {
  list-style: none;
  padding: 0;
}

.selected-files-list li {
  background-color: #f9f9f9;
  border: 1px solid #ddd;
  padding: 10px;
  margin-bottom: 5px;
  border-radius: 4px;
  font-size: 0.9em;
}

.selected-files-list li span {
  font-weight: bold;
}

.component-errors {
  margin-top: 30px;
  border-top: 1px solid #eee;
  padding-top: 20px;
  color: red;
}

.component-errors ul {
  list-style: none;
  padding: 0;
}

.component-errors li {
  background-color: #ffebeb;
  border: 1px solid #ffcccc;
  padding: 10px;
  margin-bottom: 5px;
  border-radius: 4px;
  font-size: 0.9em;
}

/* Example of overriding CSS variables */
.my-custom-uploader-class {
  --file-uploader-border-color: purple;
  --file-uploader-drag-over-border-color: darkorchid;
}

/* Example of adding custom styles via class */
.my-custom-item-class {
  box-shadow: 2px 2px 5px rgba(0, 0, 0, 0.1);
}
</style>
