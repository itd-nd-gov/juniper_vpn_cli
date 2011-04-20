require 'yaml'
require 'httpclient'
require 'highline/import'

CONF_FILE = "#{File.dirname(__FILE__)}/config.yml"
NCDIR = "#{ENV['HOME']}/.juniper_networks/network_connect"

desc "Make YAML Config"
task :makeconfig do
  ivehost = ask("What is your VPN host FQDN:  ")
  page1 = ask("What is the title page of your VPN login page? ")
  page2 = ask("What is the title page after you log in? ")

  File.open(CONF_FILE, 'w+') do |f|
    conf = {:ivehost => ivehost, :title_page1 => page1, :title_page2 => page2}
    f.write(conf.to_yaml)
  end
end

desc "Create the main() binary"
task :makebin do
  # Most of the credit for this logic goes to: http://makefile.com/.plan/2009/10/27/juniper-vpn-64-bit-linux-an-unsolved-mystery
  Dir.chdir(NCDIR)
  sh "sudo rm -f ncui"
  sh "gcc -m32 -Wl,-rpath,#{NCDIR} -o ncui libncui.so"
  sh "sudo chown root:root ncui"
  sh "sudo chmod 6711 ncui"
  conf = load_conf
  cli = HTTPClient.new
  resp = cli.get("https://#{conf[:ivehost]}")
  File.open('ssl.crt','w+') do |f|
    f.write(resp.peer_cert.to_der)
  end
end

desc "Connect to the vpn host"
task :vpn_connect do
  require 'selenium-webdriver'
  require 'selenium-client'
  user = ask("Username:  ")
  pass = ask("Password:  ") { |q| q.echo = "*" }

  conf = load_conf
  Dir.chdir(NCDIR)

  driver = Selenium::WebDriver.for :firefox
  driver.navigate.to "https://#{conf[:ivehost]}"

  begin
    sleep(1)
  end until driver.title == conf[:title_page1]

  # Fill out the form
  form =  driver.find_element(:name,'frmLogin')
  form.find_element(:name,'username').send_keys(user)
  form.find_element(:name,'password').send_keys(pass)
  form.submit

  begin
    sleep(1)
  end until driver.title == conf[:title_page2]

  cookie_str = driver.manage.all_cookies.map{|c| "#{c[:name]}=#{c[:value]}"}.join(';')

  driver.quit

  p = fork {
    exec(%Q{./ncui -h #{conf[:ivehost]} -c '#{cookie_str}' -p '' -f ssl.crt})
  }

  Process.detach(p)
  #Process.kill('HUP', p)
end

desc "Kill the Juniper VPN process"
task :vpn_disconnect do
  sh "pkill -f ncui"
end

# See the README for details
def load_conf
  YAML.load(File.read(CONF_FILE))
end

