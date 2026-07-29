require 'middleman-gh-pages'
require 'html-proofer'
require 'yaml'

TECH_DOCS_CONFIG = YAML.load_file('config/tech-docs.yml').freeze
GITHUB_REPO = TECH_DOCS_CONFIG.fetch('github_repo')
REPO_NAME = GITHUB_REPO.split('/').last

task :check_urls do
  proofer = HTMLProofer.check_directory("./build",
    {
      :check_external_hash => false,
      :ignore_missing_alt => true,
      :ignore_status_codes => [0, 401, 403, 429],
      :ignore_urls => [
        # Pull requests, branches and commits do not resolve to raw content
        %r{github\.com/(?=.*(?:pull|tree|commit))},
        # Generated at build time by the tech-docs gem; not a real page
        %r{github\.com/#{Regexp.escape(GITHUB_REPO)}/blob/.*/source/search/index\.html},
        # Pages added in a PR do not exist on the default branch yet
        %r{(?=.*#{Regexp.escape(REPO_NAME)})(?=.*github)}
      ]
    })

  token = ENV.fetch('GH_TOKEN', nil)
  proofer.before_request do |request|
    if request.base_url.include?("https://github.com/hmcts/")
      request.options[:headers]["Authorization"] = "Bearer #{token}"
      base_url_parts = request.base_url.split('/')
      # 5 parts means the URL is a bare repo, which needs a file appended to confirm it exists
      if base_url_parts.length == 5 && !request.base_url.include?('#')
        request.base_url = request.base_url.gsub("github.com", "raw.githubusercontent.com")
        request.base_url += "/#{TECH_DOCS_CONFIG.fetch('github_branch', 'main')}/README.md"
      elsif request.base_url.include?("/blob/")
        request.base_url = request.base_url.gsub("/blob", "")
        request.base_url = request.base_url.gsub("github.com", "raw.githubusercontent.com")
      end
    end
  end

  proofer.run
end
