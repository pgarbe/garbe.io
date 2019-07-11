require "rubygems"
require "tmpdir"

require "bundler/setup"
require "jekyll"
# require 'github-pages'


# Change your GitHub reponame
GITHUB_REPONAME = "pgarbe/garbe.io"

desc "Serve blog"
task :serve do
  Jekyll::Site.new(Jekyll.configuration({
    "source"      => ".",
    "destination" => "_site"
  })).render
end

desc "Generate blog files"
task :generate do
  Jekyll::Site.new(Jekyll.configuration({
    "source"      => ".",
    "destination" => "_site"
  })).process
end

task :default => [:generate]