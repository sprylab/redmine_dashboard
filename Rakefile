require 'yaml'
require 'fileutils'
require 'rubygems'
require 'bundler'

#
# Dashboard tasks
#
DEFAULT_REDMINE_VERSION = '2.4.1'

class Redmine
  attr_reader :version, :path

  def initialize(opts = {})
    @version = opts.delete(:version) { |_| ENV['REDMINE_VERSION'] || DEFAULT_REDMINE_VERSION }
    @path    = File.expand_path "./tmp/redmine/#{@version}", Dir.pwd
  end

  def downloaded?
    File.exists? File.join(path, 'Gemfile')
  end

  def database_config(env)
    {'pool' => 5, 'timeout' => 5000, 'adapter' => 'mysql2', 'database' => "rdb_#{version.gsub('.', '_')}_#{env}"}
  end

  def svn_url
    "http://svn.redmine.org/redmine/#{version == 'master' ? 'trunk' : "tags/#{version}"}"
  end

  def clean
    FileUtils.rm_rf path if File.exists? path
    FileUtils.mkdir_p path
  end

  def exec(*args)
    Dir.chdir path do
      ::Bundler.with_clean_env do
        Redmine.exec *args
      end
    end
  end

  def ruby(*args)
    exec %w(ruby -S), *args
  end

  def bx(*args)
    ruby %w(bundle exec), *args
  end

  class << self
    def exec(*args)
      STDOUT.puts "+ #{args.flatten.join ' '}"
      unless system(*args.flatten)
        raise RuntimeError, "Command failed: #{args.flatten.join ' '}"
      end
    end
  end
end

def force?
  !!ENV['FORCE']
end

RM = Redmine.new

task :default => [:setup, :spec]

desc 'Run specs.'
task :spec do |t, args|
  RM.bx 'rake', "spec[#{args}]"
end
task :ci => :spec

desc 'Setup project environment.'
task :setup => %w(redmine:install)

desc 'Start local redmine server.'
task :server => :setup do |t, args|
  RM.exec %w(rails server), args
end
task :s => %w(server)
task :clean => %w(redmine:clean)

namespace :redmine do
  desc 'Download RM.'
  task :download do
    if RM.downloaded? && !force?
      puts "Redmine #{RM.version} already downloaded. Use `redmine:clean` or FORCE=1 to force redownloaded."
    end
    if !RM.downloaded? || force?
      RM.clean
      RM.exec %w(svn export --quiet --force), RM.svn_url, '.'
      RM.exec %w(ln -s), Dir.pwd, 'plugins/redmine_dashboard'
      RM.exec %w(mkdir -p), 'public/plugin_assets'
      RM.exec %w(ln -s), File.join(Dir.pwd, 'assets'), 'public/plugin_assets/redmine_dashboard_linked'
      RM.exec %w(ln -s), File.join(Dir.pwd, 'spec'), '.'
      RM.exec %w(sed -i -e), "s/.*gem [\"']capybara[\"'].*//g", "Gemfile"
    end
  end

  desc 'Configure RM.'
  task :config => :download do
    config = {}
    if File.exists? File.join(RM.path, 'config/database.yml')
      begin
        config = YAML.load_file File.join(RM.path, 'config/database.yml')
      rescue => e
      end
    end

    config['test'] = RM.database_config(:test)
    %w(production development).each do |env|
      config[env] = RM.database_config(env) unless config[env]
    end

    File.open(File.join(RM.path, 'config/database.yml'), 'w') do |f|
      f.write YAML.dump config
    end
  end

  desc 'Install RM.'
  task :install => :config do
    unless File.exist? File.join(RM.path, '.installed') || force?
      tries = 0
      begin
        RM.exec %w(bundle install --without rmagick)
      rescue => e
        STDERR.puts 'bundle install failed. Retry...'
        retry if (tries += 1) < 5
      end
      RM.bx %w(rake generate_secret_token)
      RM.bx %w(rake db:create:all)
      RM.bx %w(rake db:migrate)

      FileUtils.touch File.join(RM.path, '.installed')
    else
      puts "Redmine #{RM.version} already installed. Use `redmine:clean` to delete redmine and reinstall or FORCE=1 to force install steps."
    end
  end

  desc 'Clean redmine directory.'
  task :clean do
    RM.clean
  end
end
