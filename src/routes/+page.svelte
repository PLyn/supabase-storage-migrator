<script>
	import { createClient } from '@supabase/supabase-js';
	import JSZip from 'jszip';

	// State
	let activeTab = $state('live');
	let oldProjectUrl = $state('');
	let oldProjectKey = $state('');
	let newProjectUrl = $state('');
	let newProjectKey = $state('');
	let uploadedZip = $state(null);
	let isProcessing = $state(false);
	let progress = $state(0);
	let totalItems = $state(0);
	let logs = $state([]);
	let error = $state('');
	let success = $state('');
	let detectedBuckets = $state([]);
	let currentOperation = $state('');

	// Derived state
	const canMigrateLive = $derived(
		oldProjectUrl && oldProjectKey && newProjectUrl && newProjectKey && !isProcessing
	);
	const canMigrateFromZip = $derived(
		uploadedZip && newProjectUrl && newProjectKey && !isProcessing
	);
	const progressPercentage = $derived(
		totalItems > 0 ? Math.round((progress / totalItems) * 100) : 0
	);

	// Helper functions
	function addLog(message, type = 'info') {
		logs = [...logs, { 
			message, 
			type, 
			timestamp: new Date().toLocaleTimeString() 
		}];
		
		// Auto-scroll to bottom
		setTimeout(() => {
			const logContainer = document.querySelector('.log-container');
			if (logContainer) {
				logContainer.scrollTop = logContainer.scrollHeight;
			}
		}, 10);
	}

	function clearLogs() {
		logs = [];
		error = '';
		success = '';
		progress = 0;
		totalItems = 0;
		currentOperation = '';
	}

	function setActiveTab(tab) {
		activeTab = tab;
		clearLogs();
	}

	async function handleZipUpload(event) {
		const file = event.target.files[0];
		if (file && file.name.endsWith('.zip')) {
			uploadedZip = file;
			addLog(`ZIP file selected: ${file.name} (${(file.size / 1024 / 1024).toFixed(2)} MB)`, 'info');
			
			// Preview ZIP contents
			try {
				const zip = new JSZip();
				const contents = await zip.loadAsync(file);
				const buckets = new Set();
				
				Object.keys(contents.files).forEach(path => {
					const parts = path.split('/');
					if (parts.length > 1 && parts[0]) {
						buckets.add(parts[0]);
					}
				});
				
				detectedBuckets = Array.from(buckets);
				if (detectedBuckets.length > 0) {
					addLog(`Detected ${detectedBuckets.length} bucket(s) in ZIP: ${detectedBuckets.join(', ')}`, 'info');
				}
			} catch (err) {
				addLog('Could not preview ZIP contents', 'warning');
			}
		} else {
			error = 'Please select a valid ZIP file';
			uploadedZip = null;
		}
	}

	function getMimeType(filename) {
		const ext = filename.split('.').pop().toLowerCase();
		const mimeTypes = {
			// Images
			'jpg': 'image/jpeg',
			'jpeg': 'image/jpeg',
			'png': 'image/png',
			'gif': 'image/gif',
			'webp': 'image/webp',
			'svg': 'image/svg+xml',
			'ico': 'image/x-icon',
			
			// Documents
			'pdf': 'application/pdf',
			'doc': 'application/msword',
			'docx': 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
			'xls': 'application/vnd.ms-excel',
			'xlsx': 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
			'ppt': 'application/vnd.ms-powerpoint',
			'pptx': 'application/vnd.openxmlformats-officedocument.presentationml.presentation',
			
			// Text
			'txt': 'text/plain',
			'csv': 'text/csv',
			'json': 'application/json',
			'xml': 'application/xml',
			'html': 'text/html',
			'css': 'text/css',
			'js': 'application/javascript',
			
			// Archives
			'zip': 'application/zip',
			'rar': 'application/x-rar-compressed',
			'7z': 'application/x-7z-compressed',
			'tar': 'application/x-tar',
			'gz': 'application/gzip',
			
			// Media
			'mp3': 'audio/mpeg',
			'wav': 'audio/wav',
			'mp4': 'video/mp4',
			'avi': 'video/x-msvideo',
			'mov': 'video/quicktime',
			'webm': 'video/webm',
			
			// Other
			'bin': 'application/octet-stream'
		};
		
		return mimeTypes[ext] || 'application/octet-stream';
	}

	async function migrateLiveProject() {
		clearLogs();
		isProcessing = true;
		currentOperation = 'Migrating from live project';

		try {
			addLog('Initializing Supabase clients...', 'info');
			
			const oldClient = createClient(oldProjectUrl, oldProjectKey);
			const newClient = createClient(newProjectUrl, newProjectKey);

			// List buckets from source project
			addLog('Fetching buckets from source project...', 'info');
			const { data: buckets, error: bucketsError } = await oldClient.storage.listBuckets();
			
			if (bucketsError) {
				throw new Error(`Failed to list buckets: ${bucketsError.message}`);
			}

			if (!buckets || buckets.length === 0) {
				addLog('No buckets found in source project', 'warning');
				success = 'Migration completed (no buckets found)';
				return;
			}

			addLog(`Found ${buckets.length} bucket(s): ${buckets.map(b => b.name).join(', ')}`, 'info');

			for (const bucket of buckets) {
				addLog(`\nProcessing bucket: ${bucket.name}`, 'info');
				
				// Create bucket in destination if it doesn't exist
				try {
					const { error: createError } = await newClient.storage.createBucket(
						bucket.name,
						{ public: bucket.public }
					);
					
					if (createError && !createError.message.includes('already exists')) {
						throw createError;
					}
					
					if (!createError) {
						addLog(`✓ Created bucket: ${bucket.name}`, 'success');
					}
				} catch (err) {
					addLog(`Bucket ${bucket.name} already exists, continuing...`, 'info');
				}

				// List and migrate files
				await migrateFilesFromBucket(oldClient, newClient, bucket.name);
			}

			success = `Migration completed! Processed ${buckets.length} bucket(s) with ${progress} total files.`;
		} catch (err) {
			error = err.message;
			addLog(`Migration failed: ${err.message}`, 'error');
		} finally {
			isProcessing = false;
		}
	}

	async function migrateFilesFromBucket(oldClient, newClient, bucketName, path = '') {
		try {
			const { data: files, error: listError } = await oldClient.storage
				.from(bucketName)
				.list(path, {
					limit: 1000,
					offset: 0
				});

			if (listError) {
				throw listError;
			}

			if (!files || files.length === 0) {
				return;
			}

			totalItems += files.length;

			for (const file of files) {
				const filePath = path ? `${path}/${file.name}` : file.name;

				if (file.metadata && file.metadata.mimetype) {
					// It's a file, download and upload
					try {
						addLog(`Downloading: ${filePath}`, 'info');
						
						const { data: fileData, error: downloadError } = await oldClient.storage
							.from(bucketName)
							.download(filePath);

						if (downloadError) {
							throw downloadError;
						}

						addLog(`Uploading: ${filePath}`, 'info');
						
						const { error: uploadError } = await newClient.storage
							.from(bucketName)
							.upload(filePath, fileData, {
								contentType: file.metadata.mimetype,
								upsert: true
							});

						if (uploadError) {
							throw uploadError;
						}

						progress += 1;
						addLog(`✓ Migrated: ${filePath}`, 'success');
					} catch (err) {
						addLog(`✗ Failed: ${filePath} - ${err.message}`, 'error');
					}
				} else {
					// It's a folder, recurse
					await migrateFilesFromBucket(oldClient, newClient, bucketName, filePath);
				}
			}
		} catch (err) {
			addLog(`Error listing files in ${bucketName}/${path}: ${err.message}`, 'error');
		}
	}

	async function migrateFromZip() {
		clearLogs();
		isProcessing = true;
		currentOperation = 'Migrating from ZIP file';

		try {
			addLog('Initializing Supabase client...', 'info');
			const newClient = createClient(newProjectUrl, newProjectKey);

			addLog('Loading ZIP file...', 'info');
			const zip = new JSZip();
			const contents = await zip.loadAsync(uploadedZip);

			// Group files by bucket
			const bucketFiles = {};
			const fileEntries = Object.entries(contents.files);
			
			fileEntries.forEach(([path, file]) => {
				if (!file.dir) {
					const parts = path.split('/');
					if (parts.length > 1) {
						const bucketName = parts[0];
						const relativePath = parts.slice(1).join('/');
						
						if (!bucketFiles[bucketName]) {
							bucketFiles[bucketName] = [];
						}
						
						bucketFiles[bucketName].push({
							path: relativePath,
							file: file
						});
					}
				}
			});

			const buckets = Object.keys(bucketFiles);
			if (buckets.length === 0) {
				throw new Error('No valid bucket structure found in ZIP file');
			}

			addLog(`Found ${buckets.length} bucket(s) in ZIP: ${buckets.join(', ')}`, 'info');
			
			// Count total files
			totalItems = Object.values(bucketFiles).reduce((sum, files) => sum + files.length, 0);
			addLog(`Total files to upload: ${totalItems}`, 'info');

			// List existing buckets
			const { data: existingBuckets } = await newClient.storage.listBuckets();
			const existingBucketNames = existingBuckets ? existingBuckets.map(b => b.name) : [];

			for (const [bucketName, files] of Object.entries(bucketFiles)) {
				addLog(`\nProcessing bucket: ${bucketName}`, 'info');
				
				// Create bucket if it doesn't exist
				if (!existingBucketNames.includes(bucketName)) {
					try {
						const { error: createError } = await newClient.storage.createBucket(
							bucketName,
							{ public: false } // Default to private
						);
						
						if (createError) {
							throw createError;
						}
						
						addLog(`✓ Created bucket: ${bucketName}`, 'success');
					} catch (err) {
						if (!err.message.includes('already exists')) {
							addLog(`✗ Failed to create bucket ${bucketName}: ${err.message}`, 'error');
							continue;
						}
					}
				}

				// Upload files
				for (const fileInfo of files) {
					try {
						const arrayBuffer = await fileInfo.file.async('arraybuffer');
						const blob = new Blob([arrayBuffer], { 
							type: getMimeType(fileInfo.path) 
						});

						// Check if file exists (optional skip)
						const { data: existingFile } = await newClient.storage
							.from(bucketName)
							.list(fileInfo.path.split('/').slice(0, -1).join('/'), {
								search: fileInfo.path.split('/').pop()
							});

						if (existingFile && existingFile.length > 0) {
							addLog(`Skipping existing: ${fileInfo.path}`, 'warning');
							progress += 1;
							continue;
						}

						const { error: uploadError } = await newClient.storage
							.from(bucketName)
							.upload(fileInfo.path, blob, {
								contentType: getMimeType(fileInfo.path),
								upsert: true
							});

						if (uploadError) {
							throw uploadError;
						}

						progress += 1;
						addLog(`✓ Uploaded: ${bucketName}/${fileInfo.path}`, 'success');
					} catch (err) {
						addLog(`✗ Failed: ${bucketName}/${fileInfo.path} - ${err.message}`, 'error');
						progress += 1;
					}
				}
			}

			success = `Migration completed! Uploaded ${progress} out of ${totalItems} files.`;
		} catch (err) {
			error = err.message;
			addLog(`Migration failed: ${err.message}`, 'error');
		} finally {
			isProcessing = false;
		}
	}
</script>

<div class="container">
	<h1>Supabase Storage Migrator</h1>
	<p class="subtitle">Migrate storage objects between Supabase projects</p>

	<div class="tabs">
		<button 
			class="tab {activeTab === 'live' ? 'active' : ''}"
			onclick={() => setActiveTab('live')}
		>
			Live Project Migration
		</button>
		<button 
			class="tab {activeTab === 'zip' ? 'active' : ''}"
			onclick={() => setActiveTab('zip')}
		>
			ZIP File Migration
		</button>
	</div>

	{#if activeTab === 'live'}
		<div class="tab-content active">
			<div class="form-section">
				<h2>Source Project</h2>
				<div class="form-group">
					<label for="old-url">Project URL</label>
					<input
						id="old-url"
						type="text"
						placeholder="https://xxx.supabase.co"
						bind:value={oldProjectUrl}
						disabled={isProcessing}
					/>
					<p class="help-text">The URL of your source Supabase project</p>
				</div>
				<div class="form-group">
					<label for="old-key">Service Role Key</label>
					<input
						id="old-key"
						type="password"
						placeholder="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
						bind:value={oldProjectKey}
						disabled={isProcessing}
					/>
					<p class="help-text">Found in Settings → API → service_role key</p>
				</div>
			</div>
		</div>
	{:else if activeTab === 'zip'}
		<div class="tab-content active">
			<div class="form-section">
				<h2>Upload Storage ZIP</h2>
				<div class="form-group">
					<label for="zip-upload" class="file-input-label {uploadedZip ? 'has-file' : ''}">
						{uploadedZip ? '✓ ZIP File Selected' : '📁 Choose ZIP File'}
					</label>
					<input
						id="zip-upload"
						type="file"
						accept=".zip"
						onchange={handleZipUpload}
						disabled={isProcessing}
					/>
					{#if uploadedZip}
						<div class="file-info">
							<div class="file-info-item">
								<strong>File:</strong> {uploadedZip.name}
							</div>
							<div class="file-info-item">
								<strong>Size:</strong> {(uploadedZip.size / 1024 / 1024).toFixed(2)} MB
							</div>
							{#if detectedBuckets.length > 0}
								<div class="bucket-list">
									<div class="file-info-item"><strong>Detected buckets:</strong></div>
									{#each detectedBuckets as bucket}
										<div class="bucket-item">
											<div class="bucket-icon">📁</div>
											<span>{bucket}</span>
										</div>
									{/each}
								</div>
							{/if}
						</div>
					{:else}
						<p class="help-text">
							ZIP file should contain bucket folders with your storage files.<br>
							Structure: bucket_name/path/to/file.ext
						</p>
					{/if}
				</div>
			</div>
		</div>
	{/if}

	<div class="form-section">
		<h2>Destination Project</h2>
		<div class="form-group">
			<label for="new-url">Project URL</label>
			<input
				id="new-url"
				type="text"
				placeholder="https://yyy.supabase.co"
				bind:value={newProjectUrl}
				disabled={isProcessing}
			/>
			<p class="help-text">The URL of your destination Supabase project</p>
		</div>
		<div class="form-group">
			<label for="new-key">Service Role Key</label>
			<input
				id="new-key"
				type="password"
				placeholder="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
				bind:value={newProjectKey}
				disabled={isProcessing}
			/>
			<p class="help-text">Found in Settings → API → service_role key</p>
		</div>
	</div>

	<div class="button-group">
		{#if activeTab === 'live'}
			<button
				class="primary-button"
				onclick={migrateLiveProject}
				disabled={!canMigrateLive}
			>
				{isProcessing ? 'Migrating...' : 'Start Live Migration'}
			</button>
		{:else}
			<button
				class="primary-button"
				onclick={migrateFromZip}
				disabled={!canMigrateFromZip}
			>
				{isProcessing ? 'Uploading...' : 'Start ZIP Migration'}
			</button>
		{/if}
	</div>

	{#if error}
		<div class="error-message">⚠️ {error}</div>
	{/if}
	
	{#if success}
		<div class="success-message">✅ {success}</div>
	{/if}

	{#if (totalItems > 0 || logs.length > 0) && !error}
		<div class="progress-container">
			{#if totalItems > 0}
				<div class="progress-text">
					Progress: {progress} / {totalItems} items ({progressPercentage}%)
				</div>
				<div class="progress-bar-wrapper">
					<div class="progress-bar" style="width: {progressPercentage}%"></div>
				</div>
			{/if}
			
			{#if logs.length > 0}
				<div class="log-container">
					{#each logs as log}
						<div class="log-entry log-{log.type}">
							<span style="opacity: 0.6">[{log.timestamp}]</span> {log.message}
						</div>
					{/each}
				</div>
			{/if}
		</div>
	{/if}
</div>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
		background: #0f0f0f;
		color: #e0e0e0;
		min-height: 100vh;
	}

	.container {
		max-width: 900px;
		margin: 0 auto;
		padding: 40px 20px;
	}

	h1 {
		font-size: 2.5rem;
		margin-bottom: 10px;
		background: linear-gradient(135deg, #3ECF8E, #38BDF8);
		-webkit-background-clip: text;
		-webkit-text-fill-color: transparent;
		background-clip: text;
	}

	.subtitle {
		color: #888;
		margin-bottom: 40px;
		font-size: 1.1rem;
	}

	.tabs {
		display: flex;
		gap: 10px;
		margin-bottom: 30px;
		border-bottom: 2px solid #333;
		padding-bottom: 10px;
	}

	.tab {
		padding: 10px 20px;
		background: none;
		border: none;
		color: #888;
		font-size: 16px;
		cursor: pointer;
		transition: all 0.2s;
		position: relative;
	}

	.tab:hover {
		color: #ccc;
	}

	.tab.active {
		color: #3ECF8E;
	}

	.tab.active::after {
		content: '';
		position: absolute;
		bottom: -12px;
		left: 0;
		right: 0;
		height: 2px;
		background: #3ECF8E;
	}

	.tab-content {
		display: block;
	}

	.form-section {
		margin-bottom: 30px;
		padding: 25px;
		background: #242424;
		border-radius: 8px;
		border: 1px solid #333;
	}

	.form-section h2 {
		font-size: 1.3rem;
		margin-bottom: 20px;
		color: #3ECF8E;
	}

	.form-group {
		margin-bottom: 20px;
	}

	label {
		display: block;
		margin-bottom: 8px;
		color: #ccc;
		font-weight: 500;
	}

	input[type="text"],
	input[type="password"] {
		width: 100%;
		padding: 12px 16px;
		background: #1a1a1a;
		border: 1px solid #444;
		border-radius: 6px;
		color: #fff;
		font-size: 16px;
		transition: all 0.2s;
	}

	input[type="text"]:focus,
	input[type="password"]:focus {
		outline: none;
		border-color: #3ECF8E;
		background: #222;
	}

	input[type="file"] {
		display: none;
	}

	.file-input-label {
		display: inline-block;
		padding: 12px 24px;
		background: #2a2a2a;
		border: 2px dashed #444;
		border-radius: 6px;
		cursor: pointer;
		transition: all 0.2s;
		color: #ccc;
		width: 100%;
		text-align: center;
		box-sizing: border-box;
	}

	.file-input-label:hover {
		border-color: #3ECF8E;
		color: #3ECF8E;
	}

	.file-input-label.has-file {
		border-color: #3ECF8E;
		background: #1a3a2a;
	}

	.file-info {
		margin-top: 10px;
		padding: 10px;
		background: #1a1a1a;
		border-radius: 4px;
		font-size: 14px;
	}

	.file-info-item {
		color: #888;
		margin-bottom: 5px;
	}

	.file-info-item strong {
		color: #ccc;
	}

	.bucket-list {
		margin-top: 15px;
		padding: 15px;
		background: #0a0a0a;
		border-radius: 4px;
		border: 1px solid #333;
	}

	.bucket-item {
		padding: 8px 12px;
		margin-bottom: 8px;
		background: #1a1a1a;
		border-radius: 4px;
		display: flex;
		align-items: center;
		gap: 10px;
	}

	.bucket-icon {
		width: 20px;
		height: 20px;
		background: #3ECF8E;
		border-radius: 4px;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 12px;
		color: white;
	}

	.button-group {
		display: flex;
		gap: 10px;
		margin-top: 30px;
	}

	button {
		flex: 1;
		padding: 14px 28px;
		border: none;
		border-radius: 6px;
		font-size: 16px;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.2s;
		position: relative;
		overflow: hidden;
	}

	.primary-button {
		background: linear-gradient(135deg, #3ECF8E, #38BDF8);
		color: white;
	}

	.primary-button:hover:not(:disabled) {
		transform: translateY(-2px);
		box-shadow: 0 10px 20px rgba(62, 207, 142, 0.3);
	}

	button:disabled {
		opacity: 0.5;
		cursor: not-allowed;
	}

	.progress-container {
		margin-top: 30px;
		padding: 20px;
		background: #1a1a1a;
		border-radius: 8px;
		border: 1px solid #333;
	}

	.progress-bar-wrapper {
		width: 100%;
		height: 8px;
		background: #333;
		border-radius: 4px;
		overflow: hidden;
		margin-bottom: 15px;
	}

	.progress-bar {
		height: 100%;
		background: linear-gradient(90deg, #3ECF8E, #38BDF8);
		transition: width 0.3s ease;
		border-radius: 4px;
	}

	.progress-text {
		color: #ccc;
		font-size: 14px;
		margin-bottom: 10px;
	}

	.log-container {
		max-height: 300px;
		overflow-y: auto;
		background: #0a0a0a;
		border: 1px solid #333;
		border-radius: 4px;
		padding: 15px;
		margin-top: 15px;
		font-family: 'SF Mono', Monaco, monospace;
		font-size: 13px;
	}

	.log-entry {
		margin-bottom: 8px;
		padding: 4px 0;
	}

	.log-success {
		color: #3ECF8E;
	}

	.log-error {
		color: #f87171;
	}

	.log-info {
		color: #60a5fa;
	}

	.log-warning {
		color: #fbbf24;
	}

	.error-message {
		background: #7f1d1d;
		border: 1px solid #dc2626;
		color: #fca5a5;
		padding: 12px 16px;
		border-radius: 6px;
		margin-top: 15px;
		font-size: 14px;
	}

	.success-message {
		background: #14532d;
		border: 1px solid #16a34a;
		color: #86efac;
		padding: 12px 16px;
		border-radius: 6px;
		margin-top: 15px;
		font-size: 14px;
	}

	.help-text {
		font-size: 13px;
		color: #666;
		margin-top: 8px;
	}

	:global(::-webkit-scrollbar) {
		width: 8px;
	}

	:global(::-webkit-scrollbar-track) {
		background: #1a1a1a;
	}

	:global(::-webkit-scrollbar-thumb) {
		background: #444;
		border-radius: 4px;
	}

	:global(::-webkit-scrollbar-thumb:hover) {
		background: #555;
	}
</style>