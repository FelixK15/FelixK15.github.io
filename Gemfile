# frozen_string_literal: true

source "https://rubygems.org"

gem "jekyll-theme-chirpy", "~> 7.3.0"

group :test do
  gem "html-proofer", "~> 3.18"
end

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Lock `http_parser.rb` gem to `v0.6.x` on JRuby builds since newer versions of the gem
# do not have a Java counterpart.
gem "http_parser.rb", "~> 0.6.0", :platforms => [:jruby]

# Lock jekyll-sass-converter to 2.x on Linux-musl
if RUBY_PLATFORM =~ /linux-musl/
  gem "jekyll-sass-converter", "~> 2.0"
end

# Open all external links in a new tab
gem 'jekyll-target-blank'
