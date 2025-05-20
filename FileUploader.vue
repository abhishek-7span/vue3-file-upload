<template>
  <div
    class="file-uploader"
    :class="[{ 'drag-over': isDragging }, uploaderClass]"
    @dragover.prevent="handleDragOver"
    @dragleave.prevent="handleDragLeave"
    @drop.prevent="handleDrop"
    @click="triggerFileInput"
    role="button"
    aria-label="Select or drag and drop files"
    tabindex="0"
    @keydown.enter="triggerFileInput"
    @keydown.space="triggerFileInput"
  >
    <input
      type="file"
      ref="fileInput"
      :multiple="multiple"
      @change="handleFileChange"
      style="display: none;"
      aria-hidden="true"
    />
    <p>Drag and drop files here or click to select</p>
    <!-- File previews will go here later -->
    <div class="file-previews" :class="previewClass">
      <!-- Slot for custom previews -->
      <slot name="file-preview" :files="selectedFiles" :removeFile="removeFile" :uploadFile="uploadFile">
        <!-- Default preview content -->
        <div v-for="(file, index) in selectedFiles" :key="file.name + index" class="file-item" :class="itemClass">
          <div class="file-preview-thumbnail">
            <img v-if="file.type.startsWith('image/')" :src="file.previewUrl" :alt="file.name" class="preview-image"/>
            <span v-else-if="file.type === 'application/pdf'" class="preview-icon">📄</span> <!-- PDF icon -->
            <span v-else class="preview-icon">📁</span> <!-- Generic file icon -->
          </div>
          <div class="file-info">
            <span class="file-name">{{ file.name }}</span>
            <span class="file-size">({{ (file.size / 1024).toFixed(2) }} KB)</span>
            <div v-if="file.status === 'uploading'" class="upload-progress" role="progressbar" :aria-valuenow="file.progress" aria-valuemin="0" aria-valuemax="100">
                <div class="progress-bar" :style="{ width: file.progress + '%' }"></div>
                <span class="progress-text">{{ file.progress.toFixed(0) }}%</span>
            </div>
            <span v-if="file.status === 'success'" class="upload-status success" aria-live="polite">Upload Successful</span>
            <span v-if="file.status === 'error'" class="upload-status error" aria-live="polite">Upload Failed</span>
          </div>
          <button
            @click.stop="removeFile(index)"
            class="remove-button"
            aria-label="Remove file: {{ file.name }}"
          >
            &times;
          </button>
          <button
            v-if="file.status === 'error'"
            @click.stop="uploadFile(file, index)"
            class="retry-button"
            aria-label="Retry upload for file: {{ file.name }}"
          >
            Retry
          </button>
        </div>
      </slot>
    </div>
    <!-- Error messages will go here later -->
    <div v-if="errors.length" class="error-messages" :class="errorClass" aria-live="assertive">
      <p v-for="(error, index) in errors" :key="index">{{ error }}</p>
    </div>
  </div>
</template>

<script>
import { ref } from 'vue';

export default {
  name: 'FileUploader',
  props: {
    multiple: {
      type: Boolean,
      default: false
    },
    // Props for validation and limits will be added later
    // maxFileSize: Number,
    // maxTotalSize: Number,
    // maxFiles: Number,
    // allowedTypes: Array,
    // allowedMimeTypes: Array,
    // validateFile: Function,
    maxFileSize: {
      type: Number,
      default: 0 // 0 means no limit
    },
    maxTotalSize: {
      type: Number,
      default: 0 // 0 means no limit
    },
    maxFiles: {
      type: Number,
      default: 0 // 0 means no limit
    },
    allowedTypes: {
      type: Array,
      default: () => [] // e.g., ['.jpg', '.png']
    },
    allowedMimeTypes: {
      type: Array,
      default: () => [] // e.g., ['image/jpeg', 'image/png']
    },
    validateFile: {
      type: Function,
      default: null // Custom validation function
    },
    uploadUrl: {
        type: String,
        required: true // The URL to upload files to
    },
    // New props for custom classes
    uploaderClass: {
        type: [String, Array, Object],
        default: ''
    },
    previewClass: {
        type: [String, Array, Object],
        default: ''
    },
    itemClass: {
        type: [String, Array, Object],
        default: ''
    },
    errorClass: {
        type: [String, Array, Object],
        default: ''
    }
  },
  setup(props, { emit }) {
    const fileInput = ref(null);
    const isDragging = ref(false);
    // Store files with preview URLs, status, and progress
    const selectedFiles = ref([]);
    const errors = ref([]); // To store validation errors

    const triggerFileInput = () => {
      fileInput.value.click();
    };

    const handleFileChange = (event) => {
      const files = event.target.files;
      processFiles(files);
      // Clear the input value to allow selecting the same file again
      event.target.value = '';
    };

    const handleDragOver = () => {
      isDragging.value = true;
    };

    const handleDragLeave = () => {
      isDragging.value = false;
    };

    const handleDrop = (event) => {
      isDragging.value = false;
      const files = event.dataTransfer.files;
      processFiles(files);
    };

    const generatePreview = (file) => {
      return new Promise((resolve) => {
        if (file.type.startsWith('image/')) {
          const reader = new FileReader();
          reader.onload = (e) => {
            resolve(e.target.result);
          };
          reader.readAsDataURL(file);
        } else {
          // For non-image files, we don't generate a URL preview
          // We'll use CSS/icons to represent them
          resolve(null);
        }
      });
    };

    const processFiles = async (files) => { // Made async to handle custom validation promise and preview generation
      errors.value = []; // Clear previous errors
      let filesArray = Array.from(files);
      let validFiles = [];
      let currentTotalSize = selectedFiles.value.reduce((sum, file) => sum + file.size, 0);

      if (!props.multiple) {
        // If not multiple, replace existing files and revoke old preview URLs
        selectedFiles.value.forEach(file => {
            if (file.previewUrl) URL.revokeObjectURL(file.previewUrl);
        });
        selectedFiles.value = [];
        currentTotalSize = 0;
        filesArray = filesArray.slice(0, 1); // Only take the first file
      }

      // Check max files limit first
      if (props.maxFiles > 0 && (selectedFiles.value.length + filesArray.length) > props.maxFiles) {
        errors.value.push(`Maximum number of files (${props.maxFiles}) exceeded.`);
        // Truncate filesArray to fit within the limit if multiple is true
        if (props.multiple) {
             filesArray = filesArray.slice(0, props.maxFiles - selectedFiles.value.length);
        } else {
            // If not multiple and already have a file, or trying to add more than 1, this is an error handled above.
            // If trying to add 1 file but maxFiles is 0, this check is skipped.
            // If maxFiles is 1 and multiple is false, this check is also handled.
        }
         if (filesArray.length === 0 && props.multiple) {
             // If after truncating, there are no files left to process, return
             return;
         }
      }


      for (const file of filesArray) {
        let fileErrors = [];

        // Validate file size
        if (props.maxFileSize > 0 && file.size > props.maxFileSize) {
          fileErrors.push(`${file.name}: Size exceeds maximum allowed size (${(props.maxFileSize / 1024 / 1024).toFixed(2)} MB).`);
        }

        // Validate file type (extension)
        if (props.allowedTypes.length > 0) {
          const fileExtension = '.' + file.name.split('.').pop().toLowerCase();
          if (!props.allowedTypes.map(ext => ext.toLowerCase()).includes(fileExtension)) {
            fileErrors.push(`${file.name}: Invalid file type. Allowed types are: ${props.allowedTypes.join(', ')}.`);
          }
        }

        // Validate MIME type
        if (props.allowedMimeTypes.length > 0) {
           if (!props.allowedMimeTypes.includes(file.type)) {
             fileErrors.push(`${file.name}: Invalid MIME type. Allowed types are: ${props.allowedMimeTypes.join(', ')}.`);
           }
        }

        // Custom validation
        if (props.validateFile) {
          try {
            const isValid = await props.validateFile(file);
            if (!isValid) {
              fileErrors.push(`${file.name}: Failed custom validation.`);
            }
          } catch (e) {
            fileErrors.push(`${file.name}: Custom validation threw an error: ${e.message}`);
          }
        }

        if (fileErrors.length === 0) {
          // Check total size limit before adding the file
          if (props.maxTotalSize > 0 && (currentTotalSize + file.size) > props.maxTotalSize) {
             errors.value.push(`${file.name}: Adding this file would exceed the total upload size limit (${(props.maxTotalSize / 1024 / 1024).toFixed(2)} MB).`);
             // Do not add this file
          } else {
             // Generate preview URL for images
             const previewUrl = await generatePreview(file);
             // Add status and progress for upload tracking
             validFiles.push({ ...file, previewUrl, status: 'pending', progress: 0 });
             currentTotalSize += file.size;
          }
        } else {
          errors.value.push(...fileErrors);
        }
      }

      // Add valid files to the selectedFiles array
      if (props.multiple) {
        selectedFiles.value = [...selectedFiles.value, ...validFiles];
      } else {
        selectedFiles.value = validFiles; // Should only contain 0 or 1 file
      }


      if (errors.value.length > 0) {
          emit('validationError', errors.value);
      }

      // Emit fileSelected only with the files that were successfully added
      emit('fileSelected', selectedFiles.value);
    };

    const removeFile = (index) => {
        if (index >= 0 && index < selectedFiles.value.length) {
            const removedFile = selectedFiles.value[index];
            // Revoke the preview URL if it exists
            if (removedFile.previewUrl) {
                URL.revokeObjectURL(removedFile.previewUrl);
            }
            selectedFiles.value.splice(index, 1);
            emit('fileRemoved', removedFile);
        }
    };

    const uploadFile = (file, index) => {
        if (!props.uploadUrl) {
            console.error("uploadUrl prop is not set.");
            errors.value.push(`${file.name}: Upload URL is not configured.`);
            selectedFiles.value[index].status = 'error';
            emit('uploadError', { file, error: "Upload URL not configured" });
            return;
        }

        // Update file status to uploading
        selectedFiles.value[index].status = 'uploading';
        selectedFiles.value[index].progress = 0; // Reset progress on retry

        const formData = new FormData();
        formData.append('file', file); // Append the actual File object

        const xhr = new XMLHttpRequest();

        // Progress tracking
        xhr.upload.addEventListener('progress', (event) => {
            if (event.lengthComputable) {
                const percent = (event.loaded / event.total) * 100;
                selectedFiles.value[index].progress = percent;
                emit('uploadProgress', { file, progress: percent });
            }
        });

        // Upload complete (success or failure)
        xhr.addEventListener('load', () => {
            if (xhr.status >= 200 && xhr.status < 300) {
                // Success
                selectedFiles.value[index].status = 'success';
                selectedFiles.value[index].progress = 100; // Ensure progress is 100% on success
                emit('uploadSuccess', { file, response: xhr.responseText });
            } else {
                // Error
                selectedFiles.value[index].status = 'error';
                selectedFiles.value[index].progress = 0; // Reset progress on error
                const errorMsg = `Upload failed with status ${xhr.status}: ${xhr.statusText}`;
                errors.value.push(`${file.name}: ${errorMsg}`);
                emit('uploadError', { file, error: errorMsg, status: xhr.status, response: xhr.responseText });
            }
        });

        // Network error
        xhr.addEventListener('error', () => {
            selectedFiles.value[index].status = 'error';
            selectedFiles.value[index].progress = 0;
            const errorMsg = 'Network error during upload.';
            errors.value.push(`${file.name}: ${errorMsg}`);
            emit('uploadError', { file, error: errorMsg });
        });

        // Upload aborted
        xhr.addEventListener('abort', () => {
             selectedFiles.value[index].status = 'error'; // Or 'aborted' if you want a distinct status
             selectedFiles.value[index].progress = 0;
             const errorMsg = 'Upload aborted.';
             errors.value.push(`${file.name}: ${errorMsg}`);
             emit('uploadError', { file, error: errorMsg });
        });


        xhr.open('POST', props.uploadUrl);
        // You might need to set headers here, e.g., for authentication
        // xhr.setRequestHeader('Authorization', 'Bearer your_token');
        xhr.send(formData);
    };


    return {
      fileInput,
      isDragging,
      selectedFiles,
      errors,
      triggerFileInput,
      handleFileChange,
      handleDragOver,
      handleDragLeave,
      handleDrop,
      removeFile, // Expose removeFile
      uploadFile, // Expose uploadFile
    };
  },
};
</script>

<style scoped>
/* CSS Variables for Theming */
:root {
  --file-uploader-border-color: #ccc;
  --file-uploader-drag-over-border-color: #007bff;
  --file-item-border-color: #eee;
  --file-item-hover-border-color: #007bff; /* New hover color */
  --file-item-hover-background-color: #f8f9fa; /* New hover background */
  --file-info-color: #333;
  --file-size-color: #666;
  --progress-bar-background: #f3f3f3;
  --progress-bar-fill-color: #4CAF50;
  --progress-text-color: #333;
  --upload-success-color: green;
  --upload-error-color: red;
  --remove-button-background: rgba(255, 0, 0, 0.7);
  --remove-button-hover-background: red;
  --remove-button-color: white;
  --retry-button-background: rgba(0, 123, 255, 0.7);
  --retry-button-hover-background: #007bff;
  --retry-button-color: white;
  --error-message-color: red;
}


.file-uploader {
  border: 2px dashed var(--file-uploader-border-color);
  padding: 20px;
  text-align: center;
  cursor: pointer;
  transition: border-color 0.3s ease;
  position: relative;
}

.file-uploader.drag-over {
  border-color: var(--file-uploader-drag-over-border-color);
}

.file-previews {
  margin-top: 20px;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
  gap: 10px;
}

.file-item {
  border: 1px solid var(--file-item-border-color);
  padding: 10px;
  word-break: break-all;
  font-size: 0.9em;
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  min-height: 100px;
  overflow: hidden;
  transition: border-color 0.3s ease, background-color 0.3s ease; /* Add transition for hover */
}

.file-item:hover {
    border-color: var(--file-item-hover-border-color);
    background-color: var(--file-item-hover-background-color);
}


.file-preview-thumbnail {
    width: 60px;
    height: 60px;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
    margin-bottom: 5px;
}

.preview-image {
    max-width: 100%;
    max-height: 100%;
    object-fit: contain;
}

.preview-icon {
    font-size: 3em;
}

.file-info {
    display: flex;
    flex-direction: column;
    align-items: center;
    font-size: 0.8em;
    width: 100%;
    margin-top: auto;
    color: var(--file-info-color); /* Use variable */
}

.file-name {
    font-weight: bold;
    word-break: break-word;
    overflow: hidden;
    text-overflow: ellipsis;
    width: 100%;
    white-space: nowrap;
}

.file-size {
    color: var(--file-size-color); /* Use variable */
}

.upload-progress {
    width: 100%;
    height: 10px;
    background-color: var(--progress-bar-background); /* Use variable */
    border-radius: 5px;
    margin-top: 5px;
    overflow: hidden;
    position: relative;
}

.progress-bar {
    height: 100%;
    background-color: var(--progress-bar-fill-color); /* Use variable */
    transition: width 0.1s ease;
}

.progress-text {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    font-size: 0.7em;
    color: var(--progress-text-color); /* Use variable */
    z-index: 1;
}

.upload-status {
    margin-top: 5px;
    font-weight: bold;
    font-size: 0.8em;
}

.upload-status.success {
    color: var(--upload-success-color); /* Use variable */
}

.upload-status.error {
    color: var(--upload-error-color); /* Use variable */
}


.remove-button {
  position: absolute;
  top: 5px;
  right: 5px;
  background: var(--remove-button-background); /* Use variable */
  color: var(--remove-button-color); /* Use variable */
  border: none;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  font-size: 12px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  line-height: 1;
  z-index: 10;
}

.remove-button:hover {
    background: var(--remove-button-hover-background); /* Use variable */
}

.retry-button {
    position: absolute;
    bottom: 5px;
    right: 5px;
    background: var(--retry-button-background); /* Use variable */
    color: var(--retry-button-color); /* Use variable */
    border: none;
    border-radius: 3px;
    padding: 2px 5px;
    font-size: 0.7em;
    cursor: pointer;
    z-index: 10;
}

.retry-button:hover {
    background: var(--retry-button-hover-background); /* Use variable */
}


.error-messages {
  margin-top: 20px;
  color: var(--error-message-color); /* Use variable */
}
</style>