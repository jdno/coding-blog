require 'digest'
require 'time'

desc 'create a new draft post'
task :post do
    title = ENV['TITLE']
    date = Date.today
    hash = Digest::SHA256.hexdigest "#{date} #{title}"
    slug = "#{date}-#{title.downcase.gsub(/[^\w]+/, '-')}"

    file = File.join(
        File.dirname(__FILE__),
        '_posts',
        slug + '.md'
    )

    File.open(file, "w") do |f|
        f << <<-EOS.gsub(/^        /, '')
        ---
        layout: post
        title: #{title}
        identifier: #{hash}
        ---

        EOS
    end

    system ("#{ENV['EDITOR']} #{file}")
end
