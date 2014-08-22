<!DOCTYPE html>
<html class="no-js">
    <head>
        <meta charset="utf-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <title>【新手向】完整的Grape项目(从配置到部署) - sennmac - SegmentFault</title>
        <meta name="description" content="">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">

        <!-- Place favicon.ico and apple-touch-icon(s) in the root directory -->

        <link rel="stylesheet" href="http://s.segmentfault.com/css/newblog/main.css?14.6.26.2">
        
        <script src="http://s.segmentfault.com/js/jquery.js?14.6.24.4"></script>
        <script src="http://s.segmentfault.com/js/bootstrap.js?14.6.24.4"></script>
        <script src="http://s.segmentfault.com/js/lib.js?14.6.24.4"></script>
        <!--[if lt IE 9]>
        <script src="http://s.segmentfault.com/js/html5shiv.js?14.6.24.4"></script>
        <script src="http://s.segmentfault.com/js/respond.js?14.6.24.4"></script>
        <![endif]-->

                        <script src="http://s.segmentfault.com/js/sticky.js?14.6.24.4"></script>
                <script src="http://s.segmentfault.com/js/highlight.js?14.6.24.4"></script>
                
                <link rel="alternate" type="application/atom+xml" title="订阅 sennmac" />
            </head>
    <body id="blog">
        <!--[if lt IE 9]>
            <p class="browsehappy">You are using an <strong>outdated</strong> browser. Please <a href="http://browsehappy.com/">upgrade your browser</a> to improve your experience.</p>
        <![endif]-->


        
    <div class="site-nav mono clearfix">
    <ol class="breadcrumb pull-left">
        <li><a href="http://segmentfault.com/">SegmentFault</a></li>
        <li><a href="http://segmentfault.com/blogs">Blog</a></li>
                <li class="active"><a href="http://blog.segmentfault.com/sennmac">sennmac</a></li>
            </ol>
    
    <ul class="breadcrumb pull-right user-nav">
            <li><a href="http://segmentfault.com/user/login">Login()</a></li>
        </ul>
</div>

<header id="blog-header">
    <div class="container">
        <div class="row">
            <div class="col-xs-12 col-md-8 blog-name">
                <h1 class="blog-logo"><a href="http://blog.segmentfault.com/sennmac">sennmac</a></h1>
                                
                                <div class="blog-description">
</div>
                            </div>
            <div class="col-md-4 hidden-xs hidden-sm offset-secondary blog-search">
                <form role="form" action="" method="">
                    <div class="form-group blog-search">
                        <label class="sr-only" for="search">搜索日志</label>
                        <input type="text" class="form-control" placeholder="关键字搜索">
                        <button class="i-search btn-search" type="button">搜索</button>
                    </div>
                </form>
            </div>
            <!-- <div class="col-md-12 blog-nav">
                <nav class="filter-tag">
                    <strong>常用标签</strong>
                    <a href="#">python</a>
                    <a href="#">mysql</a>
                    <a href="#">centos</a>
                    <a href="#">sqlite</a>
                    <a href="#">ubuntu</a>
                </nav>
            </div> -->
        </div>
    </div>
</header><!-- end #blog-header -->

<div id="blog-body">
<div class="container">
    <div class="row">
        <div id="blog-main" class="col-xs-12 col-md-8">
            <article class="post-detail" id="a-1190000000426033" data-id="1190000000426033">
                <h1 class="post-title"><a href="http://blog.segmentfault.com/sennmac/1190000000426033">【新手向】完整的Grape项目(从配置到部署)</a></h1>
                <div class="post-content fmt">
                    
<h2>本文章以代码为主,如果诸位对代码中有任何不理解的地方,非常欢迎您提出问题,非常乐意能帮助到你</h2>

<h3>整体项目选型</h3>

<p>基础框架选用<code>Grape</code>,该项目不需要输出任何html页面,只需要实现简单的http接口.<br>
ORM层使用<code>ActiveRecord</code>,存在表间关联查询,放弃使用mongodb,转用mysql,故而选择AR.<br>
Web服务器使用<code>Rainbows</code>,支持多进程.<br>
部署使用<code>capistrano</code>,最好的部署工具.</p>

<h3>项目的加载顺序</h3>

<p>0.config/rainbows.rb<br>
服务器设置</p>

<pre><code class="lang-javascript"># rainbows config
worker_processes 4
Rainbows! do
  use :ThreadSpawn
  worker_connections 100
end

# paths and things
wd          = File.expand_path('../../', __FILE__)
tmp_path    = File.join(wd, 'tmp','pids')
log_path    = File.join(wd, 'log')

Dir.mkdir(tmp_path) unless File.exist?(tmp_path)
pid_path    = File.join(tmp_path, 'rainbows.pids')
err_path    = File.join(log_path, 'rainbows.error.log')
out_path    = File.join(log_path, 'rainbows.out.log')


# If running the master process as root and the workers as an unprivileged
# user, do this to switch euid/egid in the workers (also chowns logs):
# user "unprivileged_user", "unprivileged_group"

# tell it where to be
working_directory wd

# listen on both a Unix domain socket and a TCP port,
# we use a shorter backlog for quicker failover when busy
listen 38000, :tcp_nopush =&gt; false

# nuke workers after 30 seconds instead of 60 seconds (the default)
timeout 30

# feel free to point this anywhere accessible on the filesystem
pid pid_path

# By default, the Unicorn logger will write to stderr.
# Additionally, ome applications/frameworks log to stderr or stdout,
# so prevent them from going to /dev/null when daemonized here:
stderr_path err_path
stdout_path out_path

preload_app true

before_fork do |server, worker|
  # # This allows a new master process to incrementally
  # # phase out the old master process with SIGTTOU to avoid a
  # # thundering herd (especially in the "preload_app false" case)
  # # when doing a transparent upgrade.  The last worker spawned
  # # will then kill off the old master process with a SIGQUIT.
  old_pid = "#{server.config[:pid]}.oldbin"

  if old_pid != server.pid
    begin
      sig = (worker.nr + 1) &gt;= server.worker_processes ? :QUIT : :TTOU
      Process.kill(sig, File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH
    end
  end
  #
  # Throttle the master from forking too quickly by sleeping.  Due
  # to the implementation of standard Unix signal handlers, this
  # helps (but does not completely) prevent identical, repeated signals
  # from being lost when the receiving process is busy.
  # sleep 1
end
</code></pre>

<p>1.config.ru<br>
Rainbows还是一款Rack服务器.启动就是通过加载config.ru开始.</p>

<pre><code class="lang-javascript"># This file is used by Rack-based servers to start the application.
#加载boot.rb
require ::File.expand_path('../boot',  __FILE__)
#管理AR的数据库链接,确保每次数据库链接使用完毕之后,都可正确释放掉.
use ActiveRecord::ConnectionAdapters::ConnectionManagement

logger = Logger.new("log/#{ENV["RACK_ENV"]}.log")
#设置服务器的log
use Rack::CommonLogger, logger

# 运行Api
run ::UsersApi
</code></pre>

<p>2.boot.rb<br>
设定运行环境</p>

<pre><code class="lang-javascript">require 'rubygems'

# Set rack environment
ENV['RACK_ENV'] ||= "development"

# Set up gems listed in the Gemfile.
ENV['BUNDLE_GEMFILE'] ||= File.expand_path('../Gemfile', __FILE__)
require 'bundler/setup' if File.exists?(ENV['BUNDLE_GEMFILE'])
Bundler.require(:default, ENV['RACK_ENV'])

# Set project configuration
require File.expand_path("../application", __FILE__)
</code></pre>

<p>3.application.rb<br>
设定项目相关的配置.</p>

<pre><code class="lang-javascript"><br># set database connection
require 'active_record'

ActiveRecord::Base.establish_connection 
YAML::load(File.open('config/database.yml'))[ENV["RACK_ENV"]]
ActiveSupport.on_load(:active_record) do
  self.include_root_in_json = false
  self.default_timezone = :local
  self.time_zone_aware_attributes = false
end

#load all file
%w{lib app}.each do |dir|
  Dir.glob(File.expand_path("../#{dir}", __FILE__) + '/**/*.rb').each do |file|
    require file
  end
end

# initialize log
Dir.mkdir('log') unless File.exist?('log')
case ENV["RACK_ENV"]
  when "production"
    LogSupport.logger = "log/production.log"
  else
    LogSupport.logger = "log/development.log"
    LogSupport.console = true
end
ActiveRecord::Base.logger = LogSupport.logger
</code></pre>

<h3>加入db的rake任务</h3>

<p>tasks/db.rake</p>

<pre><code class="lang-javascript">namespace :db do

  task :environment do
    type = ENV["RAILS_ENV"] || 'development' 
    ActiveRecord::Base.establish_connection YAML::load(File.open('config/database.yml'))[type]
    ActiveRecord::Base.logger = LogSupport.logger
  end

  desc "Migrate the database through scripts in db/migrate. "
  task :migrate =&gt; :environment do
    ActiveRecord::Migrator.migrate('db/migrate', ENV["VERSION"] ? ENV["VERSION"].to_i : nil)
  end

end
</code></pre>

<h3>加入Console</h3>

<p>console.rb</p>

<pre><code class="lang-javascript">require ::File.expand_path('../../boot',  __FILE__)

require 'irb'
require 'irb/completion'

IRB.start(__FILE__)
</code></pre>

<h3>添加rainbow_stopper</h3>

<p>rainbow_stopper用来关停当前的项目的服务.</p>

<pre><code class="lang-javascript">#encoding: utf-8
require 'fileutils'

current_path = File.dirname(File.dirname(File.expand_path(__FILE__)))
puts "current path: #{current_path}"

rainbows_pid = File.join(current_path, 'tmp/pids/rainbows.pids')

if File.exist?(rainbows_pid)
  pid = File.read(rainbows_pid)
  puts "Exit rainbows running on: #{pid}"
  Process.kill('QUIT', pid.to_i) rescue nil

  FileUtils.rm_rf(rainbows_pid)
  sleep 5
else
  puts "No rainbows is running due to pid: #{rainbows_pid}"
end
</code></pre>

<h3>加入capistrano部署支持</h3>

<p>config/deploy.rb</p>

<pre><code class="lang-javascript">require "bundler/capistrano"
require "rvm/capistrano"
require "capistrano/ext/multistage"

set :stages, %w{alpha production}
set :default_stage, "alpha"

set :application, "sth"
set :repository,  "http://svn.sth.com/svn/sth/trunk/"
set :svn_username, "sennmac"
set :svn_password, "0987654321"

set :scm, :subversion
# Or: `accurev`, `bzr`, `cvs`, `darcs`, `git`, `mercurial`, `perforce`, `subversion` or `none`

# if you want to clean up old releases on each deploy uncomment this:
# after "deploy:restart", "deploy:cleanup"

# if you're still using the script/reaper helper you will need
# these http://github.com/rails/irs_process_scripts

# If you are using Passenger mod_rails uncomment this:
namespace :deploy do
  task :start do
    run "cd #{current_path}; ruby script/rainbow_stopper.rb"
    run "cd #{current_path}; export RAILS_ENV=production;export RACK_ENV=production; bundle exec rainbows -D -N -E production -c config/rainbows.rb config.ru"
  end

  task :links, :roles =&gt; :app, :except =&gt; { :no_release =&gt; true } do
    run "ln -sf #{deploy_to}/shared/config/database.yml #{latest_release}/config/database.yml"
  end
end

set :rvm_ruby_string, '1.9.3-p286'

after 'deploy:finalize_update', 'deploy:links'
before 'deploy:restart',  'deploy:migrate'
</code></pre>
                </div>
                <div class="post-meta clearfix">
                    <div class="pull-left post-rank">
                        <a href="##" class="btn btn-rank vote" data-toggle="button"><span id="article-rank">1</span> 推荐</a>
                        <a href="##" class="btn btn-rank bookmark" data-toggle="button"><span>0</span> 收藏</a>
                    </div>
                    <div class="pull-right">

                                                                                                                        
                        <span class="datetime">3月6日</span>
                        <span class="comment"><a href="#comments">没有评论</a></span>
                        <span class="views">335 浏览</span>
                    </div>
                </div>
            </article><!-- end .post-detail -->

            <div id="article-comments">
    <h3 class="comment-title">0 条评论</h3>

    <div class="comment-list">
               
         
    </div>

        <div class="comment-reply comment-nologin">
        登录后才给评论，<a href="http://segmentfault.com/user/login">请先登录</a>
    </div>
    </div>

            <ul class="post-near">
                                    </ul>
        </div><!-- end #main -->
        <div id="blog-secondary" class="col-md-4 hidden-xs hidden-sm offset-secondary">
    <aside class="aside-box" id="aside-author">
        <h3 class="aside-title">文章作者</h3>
        <div class="aside-author">
            <a href="http://segmentfault.com/u/sennmac"><img src="http://sfault-avatar.b0.upaiyun.com/141/821/1418212650-1030000000424618_big64" alt="sennmac" class="pull-left"></a>
            <div class="author-status">
                <h4><a href="http://segmentfault.com/u/sennmac">sennmac</a></h4>
                1 篇文章
            </div>
                            <div class="author-bio">
                    <p>SomeWhere In Time.</p>
                    <a href="##" class="i-more"><i class="sr-only">more</i></a>
                </div>
                    </div>
    </aside>

    <aside class="aside-box" id="aside-tag-list">
        <h3 class="aside-title">文章标签</h3>
        <p class="inline-tag-list">
                    <a href="http://segmentfault.com/t/ruby" class="tag" data-tid="1040000000089699">ruby</a>
                    <a href="http://segmentfault.com/t/capistrano" class="tag" data-tid="1040000000155466">capistrano</a>
                    <a href="http://segmentfault.com/t/activerecord" class="tag" data-tid="1040000000090878">activerecord</a>
                </p>
    </aside>

    <div class="aside-pin">
        <aside class="aside-box" id="aside-share">
            <div class="aside-share">
                分享到&nbsp;
                <a href="http://segmentfault.com/share/weibo/1190000000426033" target="_blank"><i class="i-weibo">新浪微博</i></a>
                <a href="http://segmentfault.com/share/twitter/1190000000426033" target="_blank"><i class="i-twitter">Twitter</i></a>
                <a href="http://segmentfault.com/share/renren/1190000000426033" target="_blank"><i class="i-renren">人人网</i></a>
                <a href="http://segmentfault.com/share/qq/1190000000426033" target="_blank"><i class="i-tqq">腾讯微博</i></a>
                <a href="http://segmentfault.com/share/douban/1190000000426033" target="_blank"><i class="i-douban">豆瓣</i></a>
            </div>
        </aside>

                <aside class="aside-box" id="aside-recommend-post">
            <h3 class="aside-title">你可能感兴趣的</h3>
            <ul class="aside-link-list">
                                <li>
                    <a href="http://blog.segmentfault.com/fengxiuping/1190000000515219">关于smarty 模板中继承</a>
                                    </li>
                                <li>
                    <a href="http://blog.segmentfault.com/taotao123/1190000000428973">网页添加常用的分享代码，不需要第三方接口</a>
                                        <span>+1</span>
                                    </li>
                                <li>
                    <a href="http://blog.segmentfault.com/susheng/1190000000611223">JS模板引擎handlebars</a>
                                    </li>
                                <li>
                    <a href="http://blog.segmentfault.com/keysama/1190000000624669">$个人笔记['基于PHP/CURL/codeIgniter'的Spider Webbot爬虫'][2]=使用LIB_parse函数库去抓取网页中的关键字信息，并进行字符串切割</a>
                                    </li>
                                <li>
                    <a href="http://blog.segmentfault.com/xiongxin/1190000000480586">railscasts学习笔记（4-23）下</a>
                                    </li>
                                <li>
                    <a href="http://blog.segmentfault.com/linearsky/1190000000354493">关于padrino的几个大坑</a>
                                        <span>+2</span>
                                    </li>
                                <li>
                    <a href="http://blog.segmentfault.com/linearsky/1190000000355577">在Mac OS X下使用homebrew安装Emacs</a>
                                    </li>
                                <li>
                    <a href="http://blog.segmentfault.com/linearsky/1190000000359282">修改omniauth的默认配置</a>
                                    </li>
                                <li>
                    <a href="http://blog.segmentfault.com/azhi/1190000000427250">这个博客是关于Ruby和Java</a>
                                    </li>
                                <li>
                    <a href="http://blog.segmentfault.com/linearsky/1190000000428082">在rails中使用sprockets-image_compressor无损压缩图片</a>
                                        <span>+1</span>
                                    </li>
                            </ul>
        </aside>
            </div>
</div>
    </div>
</div>
</div><!-- end #blog-body -->

<footer id="blog-footer">
    <div class="container">
        <div class="row">
            <div class="col-md-12">
                <p><strong>Copyright &copy; 2011-2014 SegmentFault, Inc.</strong></p>

                <p>除特别说明外, 博客内容均采用 <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/3.0/deed.zh">CC BY-NC-SA 3.0 许可协议</a> 进行许可<br>
                京 ICP 备 12004937 号, 京公网安备 110108008332 号, 当前呈现版本 13.11.19</p>

                <nav class="foot-nav">
                    <a href="http://segmentfault.com/about">关于我们</a>
                    <a href="http://segmentfault.com/tos">服务条款</a>
                    <a href="http://segmentfault.com/faq">帮助中心</a>
                    <a href="http://0x.segmentfault.com/">问题反馈</a>
                </nav>
            </div>
        </div>
    </div>
</footer><!-- end #blog-footer -->
<a id="backtop" class="hidden-xs hidden" href="#body">回顶部</a>

<script>$(document).ready(function(){$('#session').login();$('input, textarea').placeholder();$('input.input-error,textarea.input-error').keyup(function(){$('.text-error',$(this).removeClass('input-error').parent()).remove();});$('.has-dropdown > a').click(function(){var $item=$(this).parent();$("#site-nav-btn").removeClass('active');$('.site-nav').addClass('hidden-xs');$(".has-dropdown").not($item).removeClass('active');$item.toggleClass('active');return false;});$('html').click(function(){$(".has-dropdown, #site-nav-btn").removeClass('active');$('.site-nav').addClass('hidden-xs');});$('#site-nav-btn').click(function(){$(".has-dropdown").removeClass('active');$(this).toggleClass('active');$('.site-nav').toggleClass('hidden-xs').css('left',$(this).offset().left);return false;});$(document).scroll(function(){if($(this).scrollTop()>720){$('#backtop').removeClass('hidden');}else{$('#backtop').addClass('hidden');}});$('#backtop').click(function(){$('body,html').animate({scrollTop:0});return false;})
$('#msg-link').eventPopup({url:'http://x.segmentfault.com/event'});$('.meta-tags a, .tag').tagPopup('http://segmentfault.com/api/tag','#main');$('#search .input-search').searchAutoComplete({url:'http://x.segmentfault.com/autocomplete',insertAfter:'#search',searchUrl:'http://segmentfault.com/search',askUrl:'http://segmentfault.com/ask',ask:'#top-nav a.btn-m'});$('button.close',$('.top-alert').fadeIn()).click(function(){$(this).parents('.top-alert').fadeOut(function(){$(this).remove();});return false;});$('a.msg-close',$('#msg-bar').fadeIn().sticky()).click(function(){$(this).parent().fadeOut(function(){$(this).parent().remove();});return false;});var topNav=$('.head-nav'),search=$('#search .input-search').css({'position':'relavtive','z-index':10}),searchWidth=search.outerWidth();search.focus(function(){search.animate({width:searchWidth*1.5},'fast');topNav.hide();}).blur(function(){if(0==search.val().length){search.animate({width:searchWidth},'fast',function(){topNav.show();});}});$('[rel=tooltip]').tooltip({container:'body'});if($('#profile-tab,.greeting').length>0){$.get('http://x.segmentfault.com/news',function(o){if(!o.status&&o.data){var cv=$.cookie('sfns_viewed'),viewed=(!!cv?cv+',':'');if(viewed.indexOf(o.data[0])<0){var a=$('<a href="'+o.data[1]+'" target="_blank" class="update-log rounded-2">'
+'<strong>'+o.data[3]+'更新</strong> '+o.data[2]+'</a>').prependTo('#secondary').animate({backgroundColor:'#ffebb7'},'slow',function(){$(this).animate({backgroundColor:'#FFF7E2'},'slow');}).click(function(){$.cookie('sfns_viewed',viewed+o.data[0],{path:'/',expires:30});$(this).remove();});}}},'jsonp');}
if(login){$('#main, .layout-main').highlightTag({url:'http://x.segmentfault.com/tag/following',selector:'#content article .meta-tags li a, #user-question article .meta-tags li a,'
+' .article-item .tags li a',className:'q-highlight'});}else if(0==$('.auth-login,.session-form,.session-finished').length){if(1!=$.cookie('sfln_viewed')){$.cookie('sfln_viewed',1,{path:'/'});$.cookie('sfln_available',1,{path:'/'});}else if(1==$.cookie('sfln_available')){$('.i-cancel',$('.login-notify').css('bottom',-60).removeClass('hidden').animate({'bottom':0})).click(function(){$.cookie('sfln_available',0,{path:'/'});$(this).parents('.login-notify').animate({'bottom':-60},function(){$(this).remove();})
return false;});}else if(0==$.cookie('sfln_available')){$('.login-notify').css({'padding':'10px 0','text-align':'center','bottom':-80}).html('<div class="row find-more-world">立即登录，一起讨论最纯粹的技术问题： <a class="auth-small" href="http://segmentfault.com/user/oauth/weibo"><i class="i-weibo"></i>新浪微博</a> <a class="auth-small" href="http://segmentfault.com/user/oauth/qq"><i class="i-qq"></i>腾讯QQ</a></div>').animate({'bottom':0});}}
var tagInterestShow=false;$('.tag-interest-click').click(function(){if(!tagInterestShow){$('.tag-interest-list').fadeIn();tagInterestShow=true;}else{$('.tag-interest-list').fadeOut();tagInterestShow=false;}
return false;});function hotkeySubmit1(arg){$(arg.parents).on('focus',arg.textArea,function(){var t=$(this).closest(arg.scope);Mousetrap.bind(arg.hotkey,function(){$(arg.submitBtn,t).trigger('click');});}).on('blur','.mousetrap',function(){Mousetrap.bind(arg.hotkey,function(){return false;});});}
hotkeySubmit1({parents:'.write-event-comment',textArea:'.mousetrap',scope:'form',hotkey:['command+enter','ctrl+enter'],submitBtn:'button[type = submit]'});hotkeySubmit1({parents:'.write-event-comment',textArea:'.mousetrap',scope:'form',hotkey:['command+enter','ctrl+enter'],submitBtn:'button[type = submit]'});hotkeySubmit1({parents:'.write-content',textArea:'#wmd-input',scope:'body',hotkey:['command+enter','ctrl+enter'],submitBtn:'.action'});hotkeySubmit1({parents:'#blog-main',textArea:'#comment-text',scope:'form',hotkey:['command+enter','ctrl+enter'],submitBtn:'button[type = submit]'});hotkeySubmit1({parents:'.comment-list',textArea:'.add-comment-text',scope:'.add-comment',hotkey:['command+enter','ctrl+enter'],submitBtn:'button[type = submit]'});hotkeySubmit1({parents:'#write-answer',textArea:'#wmd-input',scope:'form',hotkey:['command+enter','ctrl+enter'],submitBtn:'input[type = submit]'});hotkeySubmit1({parents:'#question',textArea:'.add-comment-text',scope:'form',hotkey:['command+enter','ctrl+enter'],submitBtn:'input[type = submit]'});hotkeySubmit1({parents:'#question',textArea:'.modify-comment-text',scope:'.add-comment',hotkey:['command+enter','ctrl+enter'],submitBtn:'.action'});hotkeySubmit1({parents:'#answer',textArea:'.add-comment-text',scope:'form',hotkey:['command+enter','ctrl+enter'],submitBtn:'input[type = submit]'});hotkeySubmit1({parents:'#answer',textArea:'.modify-comment-text',scope:'.add-comment',hotkey:['command+enter','ctrl+enter'],submitBtn:'.action'});hotkeySubmit1({parents:'#ask',textArea:'#wmd-input',scope:'#ask',hotkey:['command+enter','ctrl+enter'],submitBtn:'.ask-q-submit'});hotkeySubmit1({parents:'.edit-post',textArea:'#wmd-input',scope:'body',hotkey:['command+enter','ctrl+enter'],submitBtn:'.edit-q-submit'});var $adImgs=$('.ad-imgs');if($adImgs.length>0){$adImgs.find('.ai-item').each(function(){var $close=$(this).find('.close');if($.cookie($(this).data('adn'))==1){$(this).hide();}
$close.on('click',function(){$(this).parent().slideUp();$.cookie($(this).parent().data('adn'),1);})
$(this).on('mouseover',function(){$close.show();}).on('mouseout',function(){$close.hide();})})}
window.console&&window.console.info('   __   __________________\n _//]| | QQ 群: 206236214 |\n|____|-|__________________|  老司机~带带我~~~\n  O      OO         OO OO');window.console.info&&console.info('招聘 PHP、前端、移动端工程师 http://segmentfault.com/hiring');});(function(){var hm=document.createElement('script');hm.type='text/javascript';hm.async=true;hm.src='//hm.baidu.com/hm.js?e23800c454aa573c0ccb16b52665ac26';var s=document.getElementsByTagName('script')[0];s.parentNode.insertBefore(hm,s);})();(function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){(i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)})(window,document,'script','//www.google-analytics.com/analytics.js','ga');ga('create','UA-918487-8','segmentfault.com');ga('require','displayfeatures');ga('send','pageview');</script>
<script>(function(){if('undefined'==typeof(hljs)){return;}
hlNames={actionscript:/^as[1-3]$/i,cmake:/^(make|makefile)$/i,cs:/^csharp$/i,css:/^css[1-3]$/i,delphi:/^pascal$/i,javascript:/^js$/i,markdown:/^md$/i,objectivec:/^(oc|objective-c)$/i,php:/^php[1-6]$/i,sql:/^mysql$/i,xml:/^(html|html5|xhtml)$/i},hlLangs=hljs.LANGUAGES;SF.highlight=function(sel){var el=$(sel);$('pre',el).each(function(){if(!this.parentNode){return;}
var t=$(this),children=t.children(),highlighted=false;if(children.length>0&&'code'==children.get(0).nodeName.toLowerCase()){var className=children.attr('class');if(!!className){var classes=className.split(/\s+/);for(var i=0;i<classes.length;i++){if(0===classes[i].indexOf('lang-')){var lang=classes[i].substring(5).toLowerCase(),finalLang;if(hlLangs[lang]){finalLang=lang;}else{for(var l in hlNames){if(lang.match(hlNames[l])){finalLang=l;}}}
if(!!finalLang){var result=hljs.highlight(finalLang,children.text(),true);children.html(result.value);highlighted=true;break;}}}}}
if(!highlighted){var html=t.html();t.html(html.replace(/<\/?[a-z]+[^>]*>/ig,''));hljs.highlightBlock(this,'',false);}});};})();</script><script>$(document).ready(function(){var aside_pin_width=$(".aside-box").width(),title=$('.post-title');$(".aside-pin").sticky({topSpacing:35,bottomSpacing:200}).width(aside_pin_width);SF.highlight('.fmt');$('.fmt a').attr('target','_blank');$('.fmt').specialUrl();$('.fmt img').each(function(){var t=$(this),url=t.attr('src');if(url.search(/segmentfault/i)!=-1){t.wrap('<a target="_blank" title="点击查看原图" href="'+url+'/view"></a>');}});$('.post-meta .delete').click(function(){var p=$(this).parents('.post-detail'),cancel=$(this).data('cancel'),word=$('<p>您确认要'+(cancel?'恢复':'删除')
+'文章「<strong></strong>」吗?</p>');$('strong',word).text(title.text());word.modal({'title':cancel?'恢复确认':'删除确认','action':cancel?'恢复':'删除','data':p.data('id'),'onAction':function(c,data){$.post('http://blog.segmentfault.com/api/article',{'do':'delete','id':data,'cancel':cancel},function(o){if(!o.status){window.location.reload();}},'json');}});return false;});$('.post-meta .recommend').click(function(){var p=$(this).parents('.post-detail'),cancel=$(this).data('cancel'),word=$('<p>您确认要'+(cancel?'取消推荐':'推荐')
+'文章「<strong></strong>」吗?</p>');$('strong',word).text(title.text());word.modal({'title':cancel?'取消推荐确认':'推荐确认','action':cancel?'取消推荐':'推荐','data':p.data('id'),'onAction':function(c,data){$.post('http://blog.segmentfault.com/api/article',{'do':'recommend','id':data,'cancel':cancel},function(o){if(!o.status){window.location.reload();}},'json');}});return false;});$('.post-meta .draft').click(function(){var p=$(this).parents('.post-detail'),cancel=$(this).data('cancel'),word=$('<p>您确认要将文章「<strong></strong>」'+(cancel?'发布':'转为草稿')
+'吗?</p>');$('strong',word).text(title.text());word.modal({'title':cancel?'发布确认':'转草稿确认','action':cancel?'发布':'转草稿','data':p.data('id'),'onAction':function(c,data){$.post('http://blog.segmentfault.com/api/article',{'do':'draft','id':data,'cancel':cancel},function(o){if(!o.status){window.location.reload();}},'json');}});return false;});var comments=$('#comments');if(comments.length>0&&!!SF.param('page')){$(window).scrollTop(comments.offset().top);}
$('.comment-meta').on('click','.comment-reply-link',function(){$(this).parent().after($('.comment-reply')).append('<a href="##" class="comment-cancel-reply-link pull-right">取消回复</a>');$(this).parents('.comment-item').siblings('.comment-item').find('.comment-reply-link').show().siblings('.comment-cancel-reply-link').remove();$('.comment-reply input[name=reply]').val($(this).parents('.comment-item').data('comment').user.id);$(this).hide();$('.comment-reply textarea').focus();return false;});$('.comment-meta').on('click','.comment-cancel-reply-link',function(){$('.comment-list').after($('.comment-reply'));$(this).siblings('.comment-reply-link').show();$(this).remove();$('.comment-reply input[name=reply]').val(0);return false;});$('.comment-reply form').submit(function(){var textarea=$('textarea',this),btn=$('button',this);btn.addClass('btn-disabled').attr('disabled','disabled');$.post('http://blog.segmentfault.com/api/comment?do=post&show=page',$(this).serialize(),function(o){if(!o.status){var url=btn.data('url')+'?page='+o.data;if(window.location.href==url){window.location.reload();}else{window.location.href=url;}}else if(5==o.status){textarea.error('字数太短, 至少要包括6个字符');btn.removeAttr('disabled').removeClass('btn-disabled');}},'json');return false;});$('.btn-rank.vote').click(function(e){var t=$(this),p=t.parents('.post-detail'),id=p.data('id');$.post('http://blog.segmentfault.com/api/article',{'do':'like','id':id,'cancel':t.hasClass('active')?1:0},function(o){if(!o.status){t.toggleClass('active');$('#article-rank').html(o.data);if(t.hasClass('active')){$('.article-rank .down',p).removeClass('active');}}},'json');return false;});$('.btn-rank.bookmark').click(function(){var t=$(this),id=t.parents('.post-detail').data('id');$.post('http://blog.segmentfault.com/api/article',{'do':'bookmark','id':id,'cancel':t.hasClass('active')?1:0},function(o){if(!o.status){var el=$('span',t.toggleClass('active'));el.text(o.data);}},'json');return false;});$('#comments section a.delete').click(function(){var t=$(this).parents('section'),comment=t.data('comment');$('<p>您确认要删除来自「'+comment.user.name+'」的评论吗?</p>').modal({'title':'删除确认','action':'删除','onAction':function(c){$.post('http://blog.segmentfault.com/api/comment',{'do':'delete','id':comment.id},function(o){if(!o.status){t.fadeOut();}
c.trigger('close');},'json');}});return false;});$('#comments section a.edit').click(function(){var t=$(this).parents('section'),comment=t.data('comment'),p=$('.comment-content',t).hide(),formP=$('<div class="add-comment"></div>').insertAfter(p),contentP=$('<div class="form-group form-content" role="form"></div>').appendTo(formP),textarea=$('<textarea name="text" class="form-control mono add-comment-text mousetrap" placeholder="输入您要修改的文字" rows="1"></textarea>').appendTo(contentP).autoResize().val(comment.originalText).placeholder(),footer=$('.event-comment-footer',t).addClass('hidden'),editBtn=$('.i-edit',t).addClass('hidden');var actionP=$('<div class="form-group form-action"></div>').appendTo(formP),submitBtn=$('<button class="btn btn-primary" type="submit">保存修改</button>').appendTo(actionP),cancelBtn=$('<button class="btn btn-default">取消</button>').appendTo(actionP);submitBtn.click(function(){var btn=$(this);btn.attr('disabled','disabled').addClass('btn-disabled');$.post('http://blog.segmentfault.com/api/comment',{'do':'edit','id':comment.id,'text':textarea.val()},function(o){if(!o.status){p.empty();$(o.data.parsedText).appendTo(p);t.data('comment',o.data);p.show();formP.remove();footer.removeClass('hidden');editBtn.removeClass('hidden');}else if(5==o.status){textarea.error('字数太短, 至少要包括6个字符');btn.removeAttr('disabled').removeClass('btn-disabled');}},'json');});cancelBtn.click(function(){p.show();footer.removeClass('hidden');formP.remove();editBtn.removeClass('hidden');});$('.comment-action',t).addClass('hidden');return false;});var $imore=$('.author-bio .i-more');var $authorp=$('.author-bio p:gt(1)');if($authorp.length<1){$imore.hide();}else{$authorp.hide();$imore.on('click',function(){$authorp.toggle();return false;});}});</script>
</body>
</html>

