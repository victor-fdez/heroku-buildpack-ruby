# CAREFUL! Changes made to this file aren't tested
#
# If you need new functionality, consider putting it in lib/rake
# and also adding tests, then calling that code from here
#
require "fileutils"
require "tmpdir"
require 'hatchet/tasks'
ENV["BUILDPACK_LOG_FILE"] ||= "tmp/buildpack.log"

require_relative 'lib/rake/deploy_check'

S3_BUCKET_REGION = "us-east-1"
S3_BUCKET_NAME  = "heroku-buildpack-ruby"

def s3_tools_dir
  File.expand_path("../support/s3", __FILE__)
end

def s3_upload(tmpdir, name)
  sh("#{s3_tools_dir}/s3 put #{S3_BUCKET_NAME} #{name}.tgz #{tmpdir}/#{name}.tgz")
end

def vendor_plugin(git_url, branch = nil)
  name = File.basename(git_url, File.extname(git_url))
  Dir.mktmpdir("#{name}-") do |tmpdir|
    FileUtils.rm_rf("#{tmpdir}/*")

    Dir.chdir(tmpdir) do
      sh "git clone #{git_url} ."
      sh "git checkout origin/#{branch}" if branch
      FileUtils.rm_rf("#{name}/.git")
      sh("tar czvf #{tmpdir}/#{name}.tgz *")
      s3_upload(tmpdir, name)
    end
  end
end

def in_gem_env(gem_home, &block)
  old_gem_home = ENV['GEM_HOME']
  old_gem_path = ENV['GEM_PATH']
  ENV['GEM_HOME'] = ENV['GEM_PATH'] = gem_home.to_s

  yield

  ENV['GEM_HOME'] = old_gem_home
  ENV['GEM_PATH'] = old_gem_path
end

def install_gem(gem_name, version)
  name = "#{gem_name}-#{version}"
  Dir.mktmpdir("#{gem_name}-#{version}") do |tmpdir|
    Dir.chdir(tmpdir) do |dir|
      FileUtils.rm_rf("#{tmpdir}/*")

      in_gem_env(tmpdir) do
        sh("unset RUBYOPT; gem install #{gem_name} --version #{version} --no-ri --no-rdoc --env-shebang")
        sh("rm #{gem_name}-#{version}.gem")
        sh("rm -rf cache/#{gem_name}-#{version}.gem")
        sh("tar czvf #{tmpdir}/#{name}.tgz *")
        s3_upload(tmpdir, name)
      end
    end
  end
end

namespace :buildpack do
  desc "prepares the next version of the buildpack for release"
  task :prepare do
    puts("Use https://github.com/heroku/heroku-buildpack-ruby/actions/workflows/prepare-release.yml")
  end

  desc "releases the next version of the buildpack"
  task :release do
    deploy = DeployCheck.new(github: "heroku/heroku-buildpack-ruby")
    puts "Attempting to deploy #{deploy.next_version}, overwrite with RELEASE_VERSION env var"
    deploy.check!

    if deploy.push_tag?
      sh("git tag -f #{deploy.next_version}") do |out, status|
        raise "Could not `git tag -f #{deploy.next_version}`: #{out}" unless status.success?
      end
      sh("git push --tags") do |out, status|
        raise "Could not `git push --tags`: #{out}" unless status.success?
      end
    end

    command = "heroku buildpacks:publish heroku/ruby #{deploy.next_version}"
    puts "Releasing to heroku: `#{command}`"
    exec(command)
  end
end

desc "update plugins"
task "plugins:update" do
  vendor_plugin "https://github.com/heroku/rails_log_stdout.git", "legacy"
  vendor_plugin "https://github.com/pedro/rails3_serve_static_assets.git"
  vendor_plugin "https://github.com/hone/rails31_enable_runtime_asset_compilation.git"
end

desc "install vendored gem"
task "gem:install", :gem, :version do |t, args|
  gem     = args[:gem]
  version = args[:version]

  install_gem(gem, version)
end

begin
  require 'rspec/core/rake_task'

  desc "Run specs"
  RSpec::Core::RakeTask.new(:spec) do |t|
    t.rspec_opts = %w(-fd --color)
    #t.ruby_opts  = %w(-w)
  end
  task :default => :spec
rescue LoadError => e
end
