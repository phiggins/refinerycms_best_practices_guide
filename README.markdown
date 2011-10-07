RefineryCMS FAQ And Best Practices Guide

1. I screwed up one of the URL slugs on a site. How do I redo them?

A: You have two choices. The more severe one is to reset all slugs for that model:

    MODEL=news_item rake friendly_id:redo_slugs
    
2. When I go to /news or the URL of an engine, it tells me it can't find @page! What's going on?

A: Did you change the URL to which the page pointing to the engine redirects? If your News Page doesn't redirect to /news, then things will stop working. You can fortunately revert this, though, just by making your news page point there.

3. I need to make serious changes to an engine in my application. What's the best way to do that?

A: The best way is not to do that. If you want to take advantage of automatic updating, it's best not to change anything by hand. If you must, there are three solutions.

The first is to include your changes in the /lib/extensions folder (you'll have to create it), and then send each object the instruction to include each relevant module in the boot stanza. That looks something like this:

    # config/application.rb
    module CineflixRefinery
      class Application < Rails::Applicaiton

        # ...

        config.to_prepare do
          Dir.glob(File.join(Rails.root, "lib/extensions/**/*.rb")) do |c|
            Rails.env.production? ? require(c) : load(c)
          end
        end
        
        # ...
        
      end
    end
    
    # lib/extensions/add_press_releases_to_news_items.rb
    
    module Extensions
      module NewsItemPressReleases
        extend ActiveSupport::Concern
        included do
        
          # Model or Controller methods here
          
          scope :press_releases, lambda { where(:press_release => true) }
          scope :non_press_releases, lambda { where(:press_release => false ) }
        end
      end
    end	

    NewsItem.send(:include, Extensions::NewsItemPressReleases)
    
The advantage of this first approach is that any update to your bundled engines won't overwrite your modifications. It's minimalistically insertive, too, so if you're just looking to add some fields and methods, this is the best way to proceed.

The other two methods are more appropriate to making whole-scale changes to an engine. The first is to fork the engine on Github and then change your `Gemfile` to refer to your version. This is nice because then you can merge changes from upstream.

If you don't care about updating, though (i.e. once the site works, you won't touch it unless you are making critical security updates), another common way to do this is to check out the gem to `vendor/engines` and update your `Gemfile`. Then you have access to the entire engine structure as you would if you made your own.

To do that, on the command line, type:

    gem unpack refinerycms-news[-1.2.0] --target vendor/engines
    
And update the relevant `Gemfile` line to read:

    gem 'refinerycms-news', '1.2.0', :path => 'vendor/engines/refinerycms-news-1.2.0'
    
For convenience, you can rename the folder of the unpacked gem to something short, like 'news', so long as you update the path specified in the Gemfile.

4. How do I do integration tests? When I visit any page it shows the first time startup page because I don't have any superusers.

A. There are a few solutions you can try:

* (Recommended) Override the #show_welcome_page? method in ApplicationController in your tests. For example, if you're using RSpec you can add a file like this:

```
# in spec/support/disable_refinery_welcome_page.rb

class ApplicationController
  def show_welcome_page?
    false
  end
end
```

* Create a superuser in your tests before visiting any pages. This can be as simple as:

```
u = ::Refinery::User.new email: 'admin@example.com', username: 'admin', password: 'admin'
u.add_role(:refinery)
u.add_role(:superuser)
u.save!
```