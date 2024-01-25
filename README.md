# Test Project created using below .bat file

To run script, just run:
```
create_project --project test_project --repo test_project
```

This creates a Poetry project (installs if not already), writes my common gitignore file to the directory, and creates a Dockerfile around the poetry project.
The project is then connected to the specified repository and pushed.
The script uses a few internal or script-defined variables:
1. Project Folder: The folder in which to create the new project. In my case, this is set to my PycharmProjects folder as shown in the script.
2. Gitignore Folder: This defines where my common.gitignore file is stored. In my case, I store it in my PycharmProjects folder.
3. Git Username: In order not to define my git username in the script, I make use of `git config user.name` which has my git username stored.

One can also create the project using -P instead of --project and -R instead of --repo.

```
@echo off

rem Read GitHub username from Git configuration
for /f "delims=" %%i in ('git config user.name') do set GITHUB_USERNAME=%%i

rem Set default Projects folder name i.e. PycharmProjects
set PROJECT_FOLDER=PycharmProjects

rem Set default project path to Projects folder
set PROJECT_PATH=%USERPROFILE%\%PROJECT_FOLDER%

rem Set default location of common.gitignore file
set GITIGNORE_FOLDER=%USERPROFILE%\%PROJECT_FOLDER%

rem Parse command line arguments
:parse_args
if "%1" == "" goto end_args
if /i "%1" == "--project" (
    echo Setting PROJECT_NAME to %2
    set PROJECT_NAME=%2
    shift
    shift
) else if /i "%1" == "-P" (
    echo Setting PROJECT_NAME to %2
    set PROJECT_NAME=%2
    shift
    shift
) else if /i "%1" == "--repo" (
    echo Setting REPO_NAME to %2
    set REPO_NAME=%2
    shift
    shift
) else if /i "%1" == "-R" (
    echo Setting REPO_NAME to %2
    set REPO_NAME=%2
    shift
    shift
) else if /i "%1" == "--path" (
    set PROJECT_PATH=%2
    shift
    shift
) else (
    echo Unknown option: %1
    exit /b 1
)
goto parse_args

:end_args

if "%PROJECT_NAME%" == "" (
    echo Error: Please provide a project name using --project or -P.
    exit /b 1
)

if "%REPO_NAME%" == "" (
    echo Error: Please provide a repository name using --repo or -R.
    exit /b 1
)

set REPO_URL=https://github.com/%GITHUB_USERNAME%/%REPO_NAME%.git

rem Step 1: Poetry
rem Step 1.0: Check if Poetry is installed
where poetry >nul 2>nul
if %errorlevel% neq 0 (
    echo Installing Poetry...
    curl -sSL https://install.python-poetry.org | python -
)

rem Step 1.0.1: Change Directory to path
echo Creating project in %PROJECT_PATH% 
cd %PROJECT_PATH%

rem Step 1.1: Create a Poetry Project
poetry new %PROJECT_NAME%
if %errorlevel% neq 0 (
    echo Error: Failed to create Poetry project.
    exit /b 1
)

cd %PROJECT_NAME%

rem Step 1.2: Add default packages
rem Add black to dev packages
poetry add --group dev black

rem Step 1.3: 
(
	echo if __name__ == "__main__":
	echo ^	pass
) >> %PROJECT_NAME%/main.py



rem Step 2: Dockerfile
rem Step 2.0: Create a Dockerfile
(
	echo # The builder image, used to build the virtual environment
	echo FROM python:3.11-buster

	echo # Define system variables
	echo ## Define poetry version
	echo ENV POETRY_VERSION=1.4.0
	echo # Set the environment variable to enable non-interactive mode
	echo ENV POETRY_NO_INTERACTION=1
	echo # Poetry creates the virtual environment within the project's directory
	echo ENV POETRY_VIRTUALENVS_IN_PROJECT=1
	echo # Poetry will create a new virtual environment, even if one is already present
	echo ENV POETRY_VIRTUALENVS_CREATE=1
	echo # Poetry will use the specified directory as the cache location
	echo ENV POETRY_CACHE_DIR=/tmp/poetry_cache

	echo # Change Working Directory to /app
	echo WORKDIR /app

	echo # Copy poetry packages required to run/install
	echo COPY pyproject.toml poetry.lock README.md ./

	echo # Install packages
	echo ## Install poetry
	echo ### Install specified poetry package
	echo ## Install dependencies
	echo ### Instruct Buildkit to mount and manage a folder for caching reasons
	echo ### Avoid installing development dependencies by using --without dev
	echo ### Remove the cache once installed by using rm -rf $POETRY_CACHE_DIR
	echo ### Install dependencies before copying code by using --no-root
	echo RUN pip install "poetry==$POETRY_VERSION"
	echo #RUN --mount=type=cache,target=$POETRY_CACHE_DIR poetry install --without dev --no-root && rm -rf $POETRY_CACHE_DIR
	echo RUN poetry install --without dev --no-root && rm -rf $POETRY_CACHE_DIR

	echo COPY %PROJECT_NAME% ./%PROJECT_NAME%
	echo RUN poetry install --without dev && pip uninstall -y poetry

	echo ENTRYPOINT ["poetry", "run", "python", "-m", "%PROJECT_NAME%.main"]

	echo # Build: docker build -t "dockername" .
	echo # Run: docker run -p 9999:9999 "dockername"
) >> Dockerfile


rem Step 3: Git
rem Step 3.1: Initialize a Git Repository
git init

rem Step 3.2: Copy common.gitignore into the project
copy %GITIGNORE_FOLDER%\common.gitignore .gitignore

rem Step 3.3: Add and Commit Changes
git add .
git commit -m "Initial commit"

rem Step 3.4: Add the Remote Repository
git remote add origin %REPO_URL%
if %errorlevel% neq 0 (
    echo Error: Failed to add remote repository.
    exit /b 1
)

rem Step 3.5: Push to the Remote Repository
git push -u origin master
if %errorlevel% neq 0 (
    echo Error: Failed to push to %REPO_URL%.
    exit /b 1
)

echo Project initialized and pushed to Git repository.

```
