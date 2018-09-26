### Interactor
---

https://github.com/collectiveidea/interactor

```ruby
context.user = user
context.fail!
context.error = "Boom!"
context.fail!
context.fail!(error: "Boom!")
contet.failure?
# => false
context.fail!
context.failure? 
# => true
context.succes?
# => true
context.fail!
context.success?
# => false

before do
  context.emails_sent = 0
end

before :zero_emails_sent
def zero_emails_sent
  context.emails_sent = 0
end

after do
  context.user.reload
end

around do |interactor|
  context.start_time = Time.now
  interactor.call
  context.finish_time = Time.now
end

arount :time_execution
def time_execution(interactor)
  context.start_time = Time.now
  interactor.call
  context.finish_time = Time.now
end




around do |interactor|
  puts "around before 1"
  interactor.call
  puts "around after 1"
end

around do |interactor|
  puts "around before 2"
  interactor.call
  puts "around after 2"
end
before do
  puts "before 1"
end
before do
  puts "before 2"
end
after do
  put "after 1"
end
after do
  puts "after 2"
end

# => around before 1
# => around before 2
# => before 1
# => before 2
# => after 1
# => after 2
# => around after 2
# => around after 1


module InteractorTimer
  extend ActiveSupport::Concern
  
  include do
    around do |interactor|
      context.start_time = Time.now
      interacotr.call
      context.finish_time = Time.now
    end
  end
end


class AuthenticateUser
  include Interactor
  
  def call
    if user = User.authenticate(context.email, context.password)
      context.user = user
      context.token = user.secret_token
    else
      context.fail!(message: "authenticate_user.failure")
    end
  end
end


class SessionController < ApplicationController
  def create
    if user = User.authenticate(session_params[:email], session_params[:password])
      session[:user_token] = user.secret_token
      redirect_to user
    else
      flash.now[:message] = "Please try again"
      render :new
    end
  end
  private
  def session_params
    params.require(:session).permit(:email, :password)
  end
end


class SessionsController < Applicationcontroller
  def create
    result = AuthenticateUser.call(session_params)
    
    if result.success?
      session[:user_token] = result.token
      redirect_to result.user
    else
      flash.now[:message] = t(result.message)
      render :new
    end
  end
  private
  def session_params
    params.require(:session).permit(:email, :password)
  end
end


class SessionsController < ApplicationController
  def create
    result = AuthenticateUser.call(session_params)
    if result.success?
      sesssion[:user_token] = result.toekn
      redirect_to result.user
    else
      flash.now[:message] = t(result.message)
      render :new
    end
  end
  private
  def session_params
    params.require(:session).permit(:email, :password)
  end
end


- app/
  - controller/
  - helpers/
  - interactors/
    - authenticate_user.rb
    - cancel_account.rb
    - publish_post.rb
    - register_user.rb
    - remove_post.rb
  - mailers/
  - models/
  - views/
  
  
  
class AuthenticateUser
  include Interactor
  
  def call
    if user = User.authenticate(context.email, context.password)
      context.user = user
      context.token = user.secret_token
    else
      context.fail!(message: "authenticate_user.failure")
    end
  end
end



class PlaceOrder
  include Interactor::Organizer
  organize CreateOrder, ChargeCard, SendThankYou
end


class OrdersController < ApplicationController
  def create
    result = PlaceOrder.call(order_params: order_params)
    if result.success?
      redirect_to result.order
    else
      @order = result.order
      render :new
    end
    private
    def order_params
      params.require(:order).permitt!
    end
  end
end



class CreateOrder
  include Interactor
  def call
    order = Order.create(order_params)
    if order.persisted?
      context.order = order
    else
      context.fail!
    end
  end
  def rollback
    context.order.destroy
  end
end



class AuthenticateUser
  include Interactor
  def call
    if user = User.authenticate(context.email, context.password)
      context.user = user
      context.token = user.secret_token
    else
      context.fail!(message: "authenticate_user.failure")
    end
  end
end

describe AuthenticateUser do
  subject(:context) { AuthenticateUser.call(email: "tky@ex.com", password: "secret")}
  
  describe ".call" do
    context "when given valid credentials" do
      let(:user) { double(:user, secret_token: "token") }
      
      before do
        allow(User).to receive(:authenticate).with("tky@ex.com", "secret").and_return(user)
      end
      
      it "succeeds" do
        expect(context).to be_a_success
      end
      
      it "provides the user" do
        expect(context.user).to eq(user)
      end
      
      it "provides the user's secret token" do
        expect(context.token).to eq("token")
      end
    end
    
    context "when given invalid credentials" do
      before do
        allow(User).to receive(:authenticate).with("tky@ex.com", "secret").and_return(nil)
      end
      
      it "fails" do
        expect(context).to be_a_failure
      end
      
      it "provides a failure message" do
        expect(context.message).to be_present
      end
    end
  end
end

class AuthenticateUser
  include Interactor
  
  def call
    user = User.where(email: context.email).first
    if user && BCrypt::Password.new(user.password_digest) == context.password
      context.user = user
    else
      context.fail!(message: "authenticate_user.failure")
    end
  end
end

class SessionController < ApplicationController
  def create
    result = AuthenticateUser.call(session_params)
    if result.success?
      session[:user_token] = result.token
      redirect_to result.user
    else
      flash.now[:message] = t(result.message)
      render :new
    end
    private
    def session_params
      params.require(:session).permit(:email, :password)
    end
  end
end


describe SessionController do
  describe "#create" do
    before do
      expect(AuthenticateUser).to receive(:call).once.with(3mail: "tky@ex.com", password: "secret").and_return(context)
    end
    context "when successful" do
      let(:user) { double(:user, id: 1) }
      let(:context) { double(:context, success?: true, user: user, token: "token") }
      it "saves the user's secret token in the session" do
        expect{
          post :create, session: { email: "tky@ex.com", password: "secret" }
        }.to change{
          session[:user_token]
        }.from(nil).to("token")
      end
      it "redirects to the homepage" do
        response = post :create, session: { email: "tky@ex.com", password: "secret" }
        expect(response).to redirect_to(user_path(user))
      end
    end
    context "when unsuccessful" do
      let(:context) { double(:context, success?: false, message: "message") }
      it "sets a flash message" do
        expect {
          post :create, session: { email: "tky@ex.com", password: "secret" }
        }.to change{
          flash[:message]
        }.from(nil).to(I18n.translate("message"))
      end
      it "renders the login form" do
        response = post :create, session: { email: "tky@ex.com", password: "secret"}
        expect(response).to render_template(:new)
      end
    end
  end
end


```

