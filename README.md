### Start in 4 steps
 1. Download or clone repo
 2. Enter the folder
 3. Install Ruby gems: bundle install
 4. Start Jekyll server: jekyll serve

Access, localhost:4000

### Deploy in Github pages in 2 steps
 1. Change the variables GITHUB_REPONAME and GITHUB_REPO_BRANCH in Rakefile
 2. Run rake or rake publish for build and publish on Github

### Using Rake tasks
 * Create a new page: rake page name="contact.md"
 * Create a new post: rake post title="TITLE OF THE POST"

