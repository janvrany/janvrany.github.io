#require "stringex"

bundler='bundler3.3'

task :default => :compile 

desc "Setup (install) build tools"
task :setup do
  sh "#{bundler} install"
end

desc "Compile (generate) the site"
task :compile do
	sh "#{bundler} exec nanoc"
end

task :view do
	sh "#{bundler} exec nanoc view"
end

task :deploy do
	sh "#{bundler} exec nanoc deploy"
end

task :publish => :deploy




desc "Create a new post"
task :new_post, :title do |t, args|
  mkdir_p './content/posts'
  args.with_defaults(:title => 'New Post')
  title = args.title
  filename = "./content/posts/#{Time.now.strftime('%Y-%m-%d')}-#{title.to_url}.md"

  if File.exist?(filename)
    abort('rake aborted!') if ask("#{filename} already exists. Want to overwrite?", ['y','n']) == 'n'
  end

  puts "Creating new post: #{filename}"
  open(filename, 'w') do |post|
    post.puts '---'
    post.puts "title: \"#{title}\""
    post.puts "created_at: #{Time.now}"
    post.puts 'kind: article'
    post.puts 'published: false'
    post.puts "---\n\n"
  end
end
