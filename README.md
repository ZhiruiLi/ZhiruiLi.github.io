```bash
# Install ruby & jekyll
scoop install msys2
msys2
scoop install ruby
ridk install
gem install jekyll bundler

# Clone and start server
git clone https://github.com/ZhiruiLi/ZhiruiLi.github.io.git myblog
cd myblog
bundle exec jekyll serve
```