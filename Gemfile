source "https://rubygems.org"

# Core Jekyll
gem "jekyll", "~> 4.4.1"

# Chirpy is a remote theme
gem "jekyll-theme-chirpy"

# Required for Ruby 3.x local serve
gem "webrick"

# Ruby 4.0 will not bundle logger by default (silences the warning)
gem "logger"

group :jekyll_plugins do
  gem "jekyll-include-cache"
  gem "jekyll-paginate"
  gem "jekyll-redirect-from"
  gem "jekyll-seo-tag"
  gem "jekyll-archives"
  gem "jekyll-sitemap"
end

platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.1", platforms: [:mingw, :x64_mingw, :mswin]
gem "http_parser.rb", "~> 0.6.0", platforms: [:jruby]
