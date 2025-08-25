This is a suggested implementation plan, to be reviewed by global-planner agent.


<Orchestrator Agent Instructions: Automated Video Course Downloader - START>

I. Project Setup & Initialization

    Environment Setup:
        Create a dedicated project directory (e.g., video-course-recorder).
        Initialize a Python virtual environment within this directory.
        Install necessary Python packages:
            playwright: For headless browser automation.
            beautifulsoup4: For initial HTML parsing and general DOM traversal.
            lxml: As the preferred parser for beautifulsoup4 to enable robust XPath support.
            websocket-client: To establish and manage communication with OBS Studio via its WebSocket API.
            python-dotenv: For securely loading API keys and other sensitive configurations from a .env file.
            openai / anthropic / ollama (or similar): The specific client library for your chosen LLM API.
            json: For handling project state and parsing maps.
            requests (optional): For simple HTTP GET requests if a page is static and doesn't require a full browser.

    Directory Structure:
        ./courses/: To store state files and parsing maps for each course.
        ./logs/: To store detailed execution logs.
        ./recorded_videos/: The default output directory for OBS recordings (ensure OBS is configured to save here).

    Configuration Management:
        Create a .env file at the project root for API keys (e.g., OPENAI_API_KEY, ANTHROPIC_API_KEY).
        Create a config.json file to store general settings:
            obs_websocket_host: localhost
            obs_websocket_port: (e.g., 4444)
            obs_websocket_password: (if set in OBS)
            llm_model: (e.g., gpt-4o, claude-3-opus-20240229)
            llm_api_base_url: (if using a custom endpoint or Ollama)
            recording_buffer_seconds: 3 (for OBS timer)
            browser_profile_path: Path to the pre-authenticated browser user data directory.

II. Pre-Flight Checks (OBS & Browser)

    OBS WebSocket Connection:
        Attempt to connect to OBS Studio via the WebSocket API using websocket-client and the configured host/port/password.
        If connection fails, log an error and terminate, instructing the user to ensure OBS is running and the WebSocket plugin is active.
    OBS Settings Verification:
        Once connected, query OBS using WebSocket commands to retrieve:
            GetRecordDirectory: Confirm the recording output path matches expectations.
            GetRecordStatus: Ensure OBS is not already recording.
            GetStreamSettings (or similar): Potentially check recording format/encoder if desired for consistency.
        Display these critical settings to the user for confirmation. If settings are not ideal, prompt the user to adjust them manually in OBS.
    Browser Profile Check:
        Verify the existence and accessibility of the browser_profile_path specified in config.json. Log an error and terminate if invalid.

III. Course Processing Workflow

    User Input:
        Prompt the user to provide the main course URL (e.g., https://www.udemy.com/course/learn-cryptography-basics-in-python/learn/).

    Course State Management:
        Derive a unique course identifier from the URL (e.g., udemy.com-learn-cryptography-basics-in-python).
        Check for an existing state file: ./courses/{course_id}.json.
        If it exists, load the state (including last_processed_video_index, course_page_html_hash, course_page_parsing_map, video_page_parsing_maps).
        If not, initialize a new state file.

    Headless Browser Launch & Navigation:
        Launch Playwright (or Selenium) in headless mode, specifying the browser_profile_path to use the pre-authenticated session.
        Navigate the browser to the provided course URL.
        Wait for the page to fully load (e.g., page.wait_for_selector() for a common element, or page.wait_for_load_state('networkidle')).

    Course Page HTML & Parsing Map Acquisition:
        Retrieve the full HTML content of the course page (page.content()).
        Calculate a hash of the current HTML.
        Decision Point:
            If no course_page_parsing_map exists in the state, OR if the current HTML hash differs from course_page_html_hash in the state:
                LLM Interaction:
                    Construct a prompt for the LLM: "Analyze the following HTML to identify elements for: course title, list of video links (URL and text), module names (if applicable). Provide the output as a JSON object where keys are descriptive labels (e.g., 'course_title_xpath', 'video_links_xpath', 'video_titles_xpath', 'module_names_xpath') and values are XPath expressions. Ensure the XPath expressions are robust."
                    Send the HTML content and prompt to the chosen LLM API.
                    Parse the LLM's JSON response.
                Validation & User Confirmation:
                    Using BeautifulSoup and lxml, attempt to extract a few key pieces of information (e.g., the course title, first video link) using the LLM's suggested XPath expressions.
                    Display these extracted samples to the user and ask for confirmation: "Does this parsing map look correct? (y/n)".
                    If user confirms 'n', log the issue and allow for manual correction or re-prompting (e.g., "Please refine the prompt or manually provide XPaths.").
                Save Map: If confirmed, save the course_page_parsing_map and the course_page_html_hash to the course state file.
            Else (map exists and HTML hash matches): Load the course_page_parsing_map from the state file.

    Video List Extraction:
        Apply the course_page_parsing_map (using BeautifulSoup with lxml and find / select / xpath methods) to extract:
            Course Title.
            A list of dictionaries, each containing: video_url, video_title, module_name (if applicable).
        Store this list in the course state.

    Iterate Through Videos:
        Loop through the extracted video list, starting from last_processed_video_index if resuming.
        For each video:
            Check Completion: If the video is marked as completed: true in the state, skip it.
            Navigate to Video Page: Navigate the headless browser to video_url.
            Wait for the video player and duration elements to load.

    Video Page HTML & Parsing Map Acquisition:
        Retrieve the full HTML content of the individual video page.
        Calculate a hash of this HTML.
        Decision Point:
            Check if a video_page_parsing_map exists for this specific video page URL (or a generic one for the domain) in the state, AND if its HTML hash matches.
            If not (new page structure or HTML changed):
                LLM Interaction:
                    Construct a prompt for the LLM: "Analyze the following HTML to identify elements for: video duration (e.g., '05:30'), and the precise video title (if different from the course list title). Provide the output as a JSON object with XPath expressions (e.g., 'video_duration_xpath', 'video_title_xpath')."
                    Send the HTML content and prompt to the LLM API.
                Validation & User Confirmation:
                    Attempt to extract duration and title using LLM's XPaths.
                    Display extracted data and ask for user confirmation.
                    If 'n', log and allow correction.
                Save Map: Save the video_page_parsing_map and its HTML hash to the course state file, associating it with the video page's base URL or domain.
            Else (map exists and HTML hash matches): Load the video_page_parsing_map.

    Extract Video Details:
        Apply the video_page_parsing_map to extract:
            actual_video_duration_str: (e.g., "01:23:45"). Convert this to total seconds.
            actual_video_title: The most accurate title from the video page.
        If duration or title cannot be found, log a warning and prompt the user for manual input or guidance (e.g., "Duration not found. Please provide manually or inspect HTML.").

    OBS Recording Orchestration:
        Filename Construction: Create the output filename: {domain}-{course_title_extracted}-{module_name_extracted}-{actual_video_title_extracted}.mp4. Ensure filenames are sanitized (e.g., replace /, \ with -).
        OBS Command - Set Filename: Send OBS WebSocket command to set the recording filename.
        OBS Command - Set Output Timer: Send OBS WebSocket command to set the "Stop recording after" timer to actual_video_duration_seconds + recording_buffer_seconds.
        OBS Command - Start Recording: Send OBS WebSocket command to start the recording.
        Wait: Pause the Python script for actual_video_duration_seconds + recording_buffer_seconds.
        OBS Command - Stop Recording: Send OBS WebSocket command to stop the recording (or rely on the timer to stop it, but a manual stop command ensures it).
        Post-Recording Check:
            Verify the recorded file exists in the expected output directory.
            Check its size (e.g., not zero bytes).
            If verification fails, log a detailed error.

    State Update & Logging:
        Update the course state file: Mark the current video as completed: true. Update last_processed_video_index.
        Log the outcome for the current video (success/failure, filename, duration, any errors) to a dedicated log file (./logs/{course_id}_{video_filename}.log).

    Loop Continuation / Completion:
        Continue to the next video in the list.
        Once all videos are processed, log completion.

IV. Error Handling & Resilience

    Implement try-except blocks around all critical operations (network requests, OBS commands, LLM calls, file I/O).
    On error, log the full traceback and relevant context.
    For recoverable errors (e.g., temporary network glitch, OBS command timeout), implement a retry mechanism with exponential backoff.
    For unrecoverable errors (e.g., LLM produces unparseable JSON, OBS WebSocket disconnects permanently), terminate gracefully and prompt the user with clear instructions for manual intervention or debugging.
    The last_processed_video_index in the state file is key for resuming. The script should always check if a video has been completed before attempting to record it again.

<Orchestrator Agent Instructions: Automated Video Course Downloader - END>
