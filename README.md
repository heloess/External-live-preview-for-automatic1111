## External Live Preview for AUTOMATIC1111 Stable Diffusion web UI


https://github.com/heloess/External-live-preview-for-automatic1111/assets/96503452/b246ff09-a9bb-4130-b722-2525adfc87a5







### Installing Steps
1. Download Files
2. Edit the HTML File
3. Replace Python Files
4. Restart Stable Diffusion

### 1. Download Files
Download all files.

### 2. Edit the HTML File
In the HTML file, you will find TWO instances of the following line:

`file:///your/path/stable-diffusion-webui/outputs/saved.png`

Update these lines with your actual file path. For example:

On Windows: `file:///C:/yourpath/stable-diffusion-webui/outputs/saved.png`
On Linux `file:///home/yourpath/stable-diffusion-webui/outputs/saved.png`

Save the file and close it.

### 3. Replace your Python Files
#### Backup Existing Files
First, create backups of the existing progress.py and images.py files located in the stable-diffusion-webui/modules/ directory.

#### Replace Files
Download the new progress.py and images.py files. Replace the original files in the stable-diffusion-webui/modules/ directory with the downloaded ones.

#### Update Manually (If Needed)
If you are not using AUTOMATIC1111's v1.9.4, you may need to manually update the files instead of replacing them. I haven't tested another versions. Just in case I will show how to do it manually.

#### For images.py:
1. Open stable-diffusion-webui/modules/images.py in a text editor.
2. Search for `namegen = FilenameGenerator(p, seed, prompt, image)` using Ctrl+F.
3. Below that line, add the following code block:
   
```python
current_dir = os.getcwd()
file_path_temp = os.path.join(current_dir, "outputs", "saved.png")
image.save(file_path_temp)
```

Save the file and close it.

For progress.py:
1. Open stable-diffusion-webui/modules/progress.py in a text editor.
2. Add `import os` to the import section.
3. Replace the entire `progressapi` function with the following code:

```python
def progressapi(req: ProgressRequest):
    active = req.id_task == current_task
    queued = req.id_task in pending_tasks
    completed = req.id_task in finished_tasks

    if not active:
        textinfo = "Waiting..."
        if queued:
            sorted_queued = sorted(pending_tasks.keys(), key=lambda x: pending_tasks[x])
            queue_index = sorted_queued.index(req.id_task)
            textinfo = "In queue: {}/{}".format(queue_index + 1, len(sorted_queued))
        return ProgressResponse(active=active, queued=queued, completed=completed, id_live_preview=-1, textinfo=textinfo)

    progress = 0

    job_count, job_no = shared.state.job_count, shared.state.job_no
    sampling_steps, sampling_step = shared.state.sampling_steps, shared.state.sampling_step

    if job_count > 0:
        progress += job_no / job_count
    if sampling_steps > 0 and job_count > 0:
        progress += 1 / job_count * sampling_step / sampling_steps

    progress = min(progress, 1)

    # Check if progress is 95% or more
    if progress >= 0.95:
        return ProgressResponse(active=False, queued=False, completed=True, id_live_preview=-1, textinfo="Completed")

    elapsed_since_start = time.time() - shared.state.time_start
    predicted_duration = elapsed_since_start / progress if progress > 0 else None
    eta = predicted_duration - elapsed_since_start if predicted_duration is not None else None

    live_preview = None
    id_live_preview = req.id_live_preview

    if opts.live_previews_enable and req.live_preview:
        shared.state.set_current_image()
        if shared.state.id_live_preview != req.id_live_preview:
            image = shared.state.current_image
            if image is not None:
                buffered = io.BytesIO()

                if opts.live_previews_image_format == "png":
                    if max(*image.size) <= 256:
                        save_kwargs = {"optimize": True}
                    else:
                        save_kwargs = {"optimize": False, "compress_level": 1}
                else:
                    save_kwargs = {}

                image.save(buffered, format=opts.live_previews_image_format, **save_kwargs)
                base64_image = base64.b64encode(buffered.getvalue()).decode('ascii')
                live_preview = f"data:image/{opts.live_previews_image_format};base64,{base64_image}"
                id_live_preview = shared.state.id_live_preview

                #Saves our live previews temporarily.
                current_dir = os.getcwd()
                file_path_temp = os.path.join(current_dir, "outputs", "saved.png")
                image.save(file_path_temp)


    return ProgressResponse(active=active, queued=queued, completed=completed, progress=progress, eta=eta, live_preview=live_preview, id_live_preview=id_live_preview, textinfo=shared.state.textinfo)
```

Save the file and close it.

#### 4. Restart Stable Diffusion
After making these changes, restart Stable Diffusion. You can then open your HTML file from any desired location and start using it.
