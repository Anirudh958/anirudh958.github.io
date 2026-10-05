# frozen_string_literal: true

source "https://rubygems.org"

# Pinned exactly: `_includes/sidebar.html` and `_includes/topbar.html` override theme files
# copied from this version. When upgrading, re-copy them from the new gem and re-apply the changes.
gem "jekyll-theme-chirpy", "7.6.0"

gem "html-proofer", "~> 5.0", group: :test

platforms :windows, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.2.0", :platforms => [:windows]
