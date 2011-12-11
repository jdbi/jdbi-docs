
desc "clean up old site build"
task :clean do
  sh "rm -rf _site/*"
end

desc "rebuild site"
task :build => [:clean] do
  sh "pygmentize -V" # assure pygments is available
  sh "bundle exec jekyll"
end

desc "run jekyll server"
task :server do
  sh "bundle exec jekyll --server --auto"
end

desc "setup for building etc"
task :setup do
  sh "bundle install --path _vendor/bundle"
end

desc "push current blog to site"
task :push => [:build] do
  sh "rsync -vtr --delete _site/ lasker.skife.org:/www/jdbi.org/"
end
