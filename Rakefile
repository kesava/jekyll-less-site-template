require "rubygems"
require 'rake'
require 'yaml'

SOURCE = "."
CONFIG = {
  'version' => "0.2.0",
  'layouts' => File.join(SOURCE, "_layouts"),
  'posts' => File.join(SOURCE, "_posts"),
  'post_ext' => "md",
  'theme_package_version' => "0.1.0"
}

server_port     = "4000"      # port for preview server eg. localhost:4000

# Path configuration helper
module Twitter
  class Path
    SOURCE = "."
    Paths = {
      :layouts => "_layouts",
      :themes => "_includes/themes",
      :theme_assets => "assets/themes",
      :theme_packages => "_theme_packages",
      :posts => "_posts"
    }
    
    def self.base
      SOURCE
    end

    # build a path relative to configured path settings.
    def self.build(path, opts = {})
      opts[:root] ||= SOURCE
      path = "#{opts[:root]}/#{Paths[path.to_sym]}/#{opts[:node]}".split("/")
      path.compact!
      File.__send__ :join, path
    end
  
  end #Path
end #twitter


task :default => :less
deploy_branch  = "master"
deploy_dir     = "_heroku" 
current_dir    = File.dirname(__FILE__)

desc 'Clean up generated site'
task :clean do
  cleanup
end

desc "Generating LESS from CSS"
task :less => :build do
  current_dir = ENV['PWD']
  Dir.mkdir("#{current_dir}/_site") unless Dir.exists?("#{current_dir}/_site")
  Dir.mkdir("#{current_dir}/_site/assets/") unless Dir.exists?("#{current_dir}/_site/assets/")
  Dir.mkdir("#{current_dir}/_site/assets/css") unless Dir.exists?("#{current_dir}/_site/assets/css")
  
  puts "Generating CSS from LESS..."
  cd "_less" do
    sh "lessc styles.less > #{current_dir}/_site/assets/css/bootstrap.css"
    sh "lessc overrides.less > #{current_dir}/_site/assets/css/overrides.css"
    
  end
end

desc 'Build site with Jekyll'
task :build => :clean do
  jekyll
end

desc 'Start server with --auto'
task :server => :less do
  less
  jekyll('--server --auto')
end

desc 'Build and deploy'
task :deploy => :build do
  sh 'rsync -rtzh --progress --delete _site/ username@servername:/var/www/websitename/'
end

desc 'Check links for site already running on localhost:4000'
task :check_links do
  begin
    require 'anemone'
    root = 'http://localhost:4000/'
    Anemone.crawl(root, :discard_page_bodies => true) do |anemone|
      anemone.after_crawl do |pagestore|
        broken_links = Hash.new { |h, k| h[k] = [] }
        pagestore.each_value do |page|
          if page.code != 200
            referrers = pagestore.pages_linking_to(page.url)
            referrers.each do |referrer|
              broken_links[referrer] << page
            end
          end
        end
        broken_links.each do |referrer, pages|
          puts "#{referrer.url} contains the following broken links:"
          pages.each do |page|
            puts "  HTTP #{page.code} #{page.url}"
          end
        end
      end
    end

  rescue LoadError
    abort 'Install anemone gem: gem install anemone'
  end
end


# Usage: rake post title="A Title"
desc "Begin a new post in #{CONFIG['posts']}"
task :post do
  abort("rake aborted: '#{CONFIG['posts']}' directory not found.") unless FileTest.directory?(CONFIG['posts'])
  title = ENV["title"] || "new-post"
  slug = title.downcase.strip.gsub(' ', '-').gsub(/[^\w-]/, '')
  filename = File.join(CONFIG['posts'], "#{Time.now.strftime('%Y-%m-%d')}-#{slug}.#{CONFIG['post_ext']}")
  if File.exist?(filename)
    abort("rake aborted!") if ask("#{filename} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
  end
  
  puts "Creating new post: #{filename}"
  open(filename, 'w') do |post|
    post.puts "---"
    post.puts "layout: post"
    post.puts "title: \"#{title.gsub(/-/,' ')}\""
    post.puts "category: "
    post.puts "tags: []"
    post.puts "---"
    post.puts "{% include twitter/setup %}"
  end
end # task :post

# Usage: rake page name="about.html"
# You can also specify a sub-directory path.
# If you don't specify a file extention we create an index.html at the path specified
desc "Create a new page."
task :page do
  name = ENV["name"] || "new-page.md"
  filename = File.join(SOURCE, "#{name}")
  filename = File.join(filename, "index.html") if File.extname(filename) == ""
  title = File.basename(filename, File.extname(filename)).gsub(/[\W\_]/, " ").gsub(/\b\w/){$&.upcase}
  if File.exist?(filename)
    abort("rake aborted!") if ask("#{filename} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
  end
  
  mkdir_p File.dirname(filename)
  puts "Creating new page: #{filename}"
  open(filename, 'w') do |post|
    post.puts "---"
    post.puts "layout: page"
    post.puts "title: \"#{title}\""
    post.puts "---"
    post.puts "{% include twitter/setup %}"
  end
end # task :page



def cleanup
  sh 'rm -rf _site'
end

def jekyll(opts = '')
  sh 'jekyll ' + opts
end

def less(opts = '')
  current_dir = ENV['PWD']
  Dir.mkdir("#{current_dir}/_site") unless Dir.exists?("#{current_dir}/_site")
  Dir.mkdir("#{current_dir}/_site/assets/") unless Dir.exists?("#{current_dir}/_site/assets/")
  Dir.mkdir("#{current_dir}/_site/assets/css") unless Dir.exists?("#{current_dir}/_site/assets/css")
  
  puts "Generating CSS from LESS..."
  cd "_less" do
    sh "lessc styles.less > #{current_dir}/_site/assets/css/bootstrap.css"
  end
end

desc "deploy basic rack app to heroku"
multitask :heroku do
  puts "## Deploying to Heroku "
  (Dir["#{deploy_dir}/public/*"]).each { |f| rm_rf(f) }
  system "cp -R _site/* #{deploy_dir}/public"
  puts "\n## copying _site to #{deploy_dir}/public"
  cd "#{deploy_dir}" do
    system "git add ."
    system "git add -u"
    puts "\n## Committing: Site updated at #{Time.now.utc}"
    message = "Site updated at #{Time.now.utc}"
    system "git commit -m '#{message}'"
    puts "\n## Pushing generated #{deploy_dir} website"
    system "git push heroku #{deploy_branch}"
    puts "\n## Heroku deploy complete"
  end
end

desc "preview the site in a web browser"
task :preview do
  puts "Starting to watch source with Jekyll and Less. Starting Rack on port #{server_port}"
  system "compass compile --css-dir #{source_dir}/stylesheets" unless File.exist?("#{source_dir}/stylesheets/screen.css")
  jekyllPid = Process.spawn({"OCTOPRESS_ENV"=>"preview"}, "jekyll --auto")
  compassPid = Process.spawn("compass watch")
  rackupPid = Process.spawn("rackup --port #{server_port}")

  trap("INT") {
    [jekyllPid, compassPid, rackupPid].each { |pid| Process.kill(9, pid) rescue Errno::ESRCH }
    exit 0
  }

  [jekyllPid, compassPid, rackupPid].each { |pid| Process.wait(pid) }
end
