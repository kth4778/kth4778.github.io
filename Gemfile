# frozen_string_literal: true

source "https://rubygems.org"

gem "jekyll", "~> 4.3"

# Chirpy가 gemspec에서 선언해주던 플러그인들 — 직접 선언한다
gem "jekyll-archives", "~> 2.2"
gem "jekyll-seo-tag", "~> 2.8"
gem "jekyll-sitemap", "~> 1.4"

gem "html-proofer", "~> 5.0", group: :test

platforms :windows, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.2.0", :platforms => [:windows]
