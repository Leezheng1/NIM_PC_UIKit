# 网易云信 UI组件 使用说明

## UI组件介绍

`ui_kit` 是可以帮助用户快速打造出聊天功能的UI组件，开发者可以通过一些简洁的代码，快速的实现聊天界面、最近联系人、联系人列表等功能，并实现基础的一些定制化开发。   
`ui_kit` 完全开源，如果开发者希望修改界面，只需要通过替换界面资源，修改XML配置等方式即可实现。如果开发者希望更深层次的自定义，也可自行修改代码。  
`ui_kit`提供的功能模块：聊天窗口、最近会话列表、好友列表、群组列表、个人资料、群资料。其他功能有：添加好友、黑名单管理、图片预览图、音频视频采集与播放。


## UI组件使用说明

* `ui_kit` 依赖云信PC端基础库和其他第三方库，在使用的时候需要引入多个静态库以及第三方库的头文件。
* 工程配置示例请参考 `doc/demo_config.sln` 解决方案。（此解决方案仅用于参考）
* 具体使用范例请参考 [NIM Demo For PC](https://github.com/netease-im/NIM_PC_Demo)。

## UI组件目录介绍

* `bin`目录：UI组件依赖的DLL、界面图片资源、界面布局配置
* `third_party`目录：UI组件依赖的第三方库的头文件
* `libs`目录：UI组件依赖的第三方库的静态编译文件
* `tool_kits`目录：UI组件依赖的基础库。
* `doc`目录：UI组件帮助文档

## UI组件文档

### 基础准备

* UI组件以静态库的形式提供，首先将UI组件及其依赖的基础库导入到您的解决方案中。
<img src="./doc/import_ui_kit.jpg" width="337" height="166" />

* UI组件及其依赖的库有：
	* base
	* nim_cpp_sdk（将C接口的sdk封装为C++接口）
	* db
	* duilib
	* net
	* shared
	* image_view
	* capture_image
	* curl
	* jsoncpp
	* libuv
	* openssl
	* tinyxml
	* wxsqlite3

* 由于UI组件依赖了base库的多线程框架和duilib的界面框架，所以您的主程序也需要使用相同的框架才可以顺利使用UI组件。这里说明一下您需要完成的一些步骤：

   * 继承`nbase::FrameworkThread`类并实现一个主线程类来作为UI线程，一个基本的主线程类如下： 

	```	 
	class MainThread :
	public nbase::FrameworkThread
	{
	public:
		MainThread() : nbase::FrameworkThread("MainThread") {}
		virtual ~MainThread() {}
	private:
	
		/**
		* 虚函数，初始化主线程
		* @return void	无返回值
		*/
		virtual void Init() override;
	
		/**
		* 虚函数，主线程退出时，做一些清理工作
		* @return void	无返回值
		*/
		virtual void Cleanup() override;
		
	};
	```

   * 在`wWinMain`程序入口初始化云信sdk和UI组件并且创建主线程，一个基本的程序入口如下：  
  

	```
	int WINAPI wWinMain(HINSTANCE hInst, HINSTANCE hPrevInst, LPWSTR lpszCmdLine, int nCmdShow)
	{
		nbase::AtExitManager at_manager;
	
		HRESULT hr = ::OleInitialize(NULL);
		if (FAILED(hr))
			return 0;
	
		// 初始化云信sdk和UI组件
		// your codes
	
		{
			MainThread thread; // 创建主线程
			thread.RunOnCurrentThreadWithLoop(nbase::MessageLoop::kUIMessageLoop); // 执行主线程循环
		}
	
	
		// 程序结束之前，清理云信sdk和UI组件
		// your codes
	
		::OleUninitialize();
	
		return 0;
	}
	```

	* `MainThread`类在执行后会自动开始消息循环并调用`Init`函数，在`MainThread`类的`Init`函数里,我们可以初始化程序界面库相关设置，并且启动登录窗口来正式开启界面操作。基本的`Init`和`CleanUp`函数定义如下：  
  
	
	```
	void MainThread::Init()
	{
		//注册UI线程
		nbase::ThreadManager::RegisterThread(kThreadUI);
		
		//设置Duilib的资源资源目录
		std::wstring theme_dir = QPath::GetAppPath();
		ui::GlobalManager::Startup(theme_dir + L"themes\\default", ui::CreateControlCallback());
	
		//设置程序的图标
		nim_ui::UserConfig::GetInstance()->SetIcon(IDI_ICON);
	
		//打开登陆窗口
		nim_ui::WindowsManager::SingletonShow<LoginForm>(LoginForm::kClassName);
	}
	
	void MainThread::Cleanup()
	{
		ui::GlobalManager::Shutdown();
	
		SetThreadWasQuitProperly(true);
		nbase::ThreadManager::UnregisterThread();
	}
	```

* 完成您自己的登陆窗口、主窗口等，您自定义的窗口需要基于`云信DuiLib库`，关于`云信DuiLib库`的使用方法和注意事项，请参考：[云信Duilib](./doc/nim_duilib.md)、[云信Duilib布局指南](./doc/nim_duilib_layout.md)  

### 初始化UI组件

* UI组件工程的所有功能类都在`tool_kits/ui_component/ui_kit/export`目录中的各个头文件中导出，在您的工程里只需要包含`nim_ui_all.h`头文件就可以使用UI组件的所有功能。

```
#include "ui_component/ui_kit/export/nim_ui_all.h"
```

* UI组件导出的接口类都位于`nim_ui`命名空间下,UI组件内部的各种窗口(会话窗口、黑名单窗口等)都位于`nim_comp`命名空间下。所有导出类都是单例。
* UI组件的初始化和清理接口位于`InitManager`类中。
在`wWinMain`程序入口初始化UI组件的示例如下：

```
int WINAPI wWinMain(HINSTANCE hInst, HINSTANCE hPrevInst, LPWSTR lpszCmdLine, int nCmdShow)
{
	// your codes

	// 初始化UI组件
	nim_ui::InitManager::GetInstance()->InitUiKit();

	// 创建主线程

	// 程序结束之前，清理UI组件
	nim_ui::InitManager::GetInstance()->CleanupUiKit();

	// your codes

	return 0;
}
```

### 窗口管理

UI组件提供了窗口管理类，UI组件的窗口管理接口位于`WindowsManager`类中。UI组件的各个窗口全部使用窗口管理类来打开。我们建议但并非强求您也使用窗口管理类。使用窗口管理类可以获得以下好处：

* 统一的管理和维护所有使用窗口管理类创建的窗口（包括UI组件的窗口和您自己开发的窗口）。
* 统一销毁所有使用窗口管理类创建的窗口。
* 随时获取使用窗口管理类创建的窗口指针。
* 唯一的打开某个窗口。

如果您打算让自己开发的窗口可以被窗口管理类维护，那么您的窗口类需要继承UI组件中的`nim_comp::WindowEx`类，并实现一些特定的接口，示例如下（具体实现请参考：[云信Duilib](./doc/nim_duilib.md)）：

```
class LoginForm : public nim_comp::WindowEx
{
public:
	LoginForm();
	~LoginForm();

	virtual std::wstring GetSkinFolder() override;
	virtual std::wstring GetSkinFile() override;
	virtual std::wstring GetWindowClassName() const override;

	virtual void InitWindow() override;

	virtual UINT GetClassStyle() const override;
	virtual ui::Control* CreateControl(const std::wstring& pstrClass) override;
	
	virtual std::wstring GetWindowId() const override;
}
```

* 使用窗口管理类唯一的打开某个窗口：

```
窗口类 *form = nim_ui::WindowsManager::SingletonShow<窗口类>(窗口类::kClassName);
```

* 根据窗口类名和id获取窗口指针：

```
nim_comp::WindowEx* form = nim_ui::WindowsManager::GetInstance()->GetWindow(窗口类名, 窗口ID);
```
* 获取所有窗口:

```
 nim_comp::WindowList form_list = nim_ui::WindowsManager::GetInstance()->GetAllWindows();
```
* 获取指定class对应的所有窗口

```
 nim_comp::WindowList form_list = nim_ui::WindowsManager::GetInstance()->GetWindowsByClassNam(窗口类名);
```
* 关闭所有窗口

```
 nim_ui::WindowsManager::GetInstance()->DestroyAllWindows();
```

如果您不打算使用UI组件提供的窗口管理类来管理自己的窗口，我们建议您自己实现另外的窗口管理类来管理自己的窗口。


### 登陆窗口行为管理

UI组件会帮助开发者完成登录的功能，您需要在自己的登陆窗口里完成几个回调函数来让UI组件控制您的登陆窗口。UI组件的登录管理接口位于`LoginManager`类中。您需要完成的回调函数有：

* 通知登录错误并返回错误原因的回调函数。
* 通知取消登陆的回调函数。
* 通知隐藏登陆窗口的回调函数。
* 通知销毁登陆窗口的回调函数。
* 通知显示主窗口的回调函数。

在登陆窗口初始化时调用`LoginManager`类的`RegLoginManagerCallback`函数来注册回调函数。示例如下：

```
void LoginForm::RegLoginManagerCallback()
{
	nim_ui::OnLoginError cb_result = [this](int error){
		this->OnLoginError(error);
	};

	nim_ui::OnCancelLogin cb_cancel = [this]{
		this->OnCancelLogin();
	};

	nim_ui::OnHideWindow cb_hide = [this]{
		this->ShowWindow(false, false);
	};

	nim_ui::OnDestroyWindow cb_destroy = [this]{
		::DestroyWindow(this->GetHWND());
	};

	nim_ui::OnShowMainWindow cb_show_main = [this]{
		nim_ui::WindowsManager::SingletonShow<MainForm>(MainForm::kClassName);
	};

	nim_ui::LoginManager::GetInstance()->RegLoginManagerCallback(ToWeakCallback(cb_result),
		ToWeakCallback(cb_cancel),
		ToWeakCallback(cb_hide),
		ToWeakCallback(cb_destroy),
		ToWeakCallback(cb_show_main));
}
```

### 集成最近会话列表

最近会话列表组件位于`SessionListManager`类中。在您需要集成最近会话列表的窗口类中，为其对应的XML布局中增加一个`VListBox`控件，在窗口类初始化时获取对应`VListBox`控件的指针，调用`SessionListManager`类的`AttachListBox`函数，让UI组件依附到您的`VListBox`控件中。之后调用`InvokeLoadSessionList`函数来加载之前保存的最近会话列表项，调用`QueryUnreadCount`函数来获取每个最近会话列表项中的未读消息数，示例代码如下：

```
void MainForm::InitWindow()
{
	ui::ListBox* session_list = (ListBox*)FindControl(L"session_list");
	nim_ui::SessionListManager::GetInstance()->AttachListBox(session_list);

	nim_ui::SessionListManager::GetInstance()->InvokeLoadSessionList();
	nim_ui::SessionListManager::GetInstance()->QueryUnreadCount();

}
```

UI组件会自动在适当的时候最近会话列表的顶端位置显示`消息中心按钮`和`多端登录按钮`。单击`消息中心按钮`会打开`消息中心窗口`，其中包含`系统通知消息列表`和`自定义消息通知列表`；单击`多端登录按钮`会打开`多端登录管理窗口`,可以看到其他端是否同时登录了云信，并且可以踢出其他端。

当集成最近会话列表的窗体销毁时，应该解除对`VListBox`控件的依附，解除代码如下：

```
nim_ui::SessionListManager::GetInstance()->AttachListBox(nullptr);
```

### 集成好友列表

好友列表组件位于`ContactsListManager`类中。在您需要集成好友列表的窗口类中，为其对应的XML布局中增加一个`TreeView`控件，在窗口类初始化时获取对应`TreeView`控件的指针，调用`ContactsListManager`类的`AttachFriendListBox`函数，让UI组件依附到您的`TreeView`控件中。之后调用`InvokeGetAllUserInfo`函数来加载好友列表项，示例代码如下：

```
void MainForm::InitWindow()
{
	ui::TreeView* friend_list = (TreeView*) FindControl(L"friend_list");
	nim_ui::ContactsListManager::GetInstance()->AttachFriendListBox(friend_list);

	nim_ui::ContactsListManager::GetInstance()->InvokeGetAllUserInfo();

}
```

UI组件会自动好友列表的顶端位置显示`添加好友按钮`和`黑名单按钮`。单击`添加好友按钮`会打开`添加好友窗口`，单击`黑名单按钮`会打开`黑名单管理窗口`。

当集成好友列表的窗体销毁时，应该解除对`TreeView`控件的依附，解除代码如下：

```
nim_ui::ContactsListManager::GetInstance()->AttachFriendListBox(nullptr);
```

### 集成群组列表

群组列表组件位于`ContactsListManager`类中。在您需要集成群组列表的窗口类中，为其对应的XML布局中增加一个`TreeView`控件，在窗口类初始化时获取对应`TreeView`控件的指针，调用`ContactsListManager`类的`AttachGroupListBox`函数，让UI组件依附到您的`TreeView`控件中。群组组件会自动加载群列表项，示例代码如下：

```
void MainForm::InitWindow()
{
	ui::TreeView* group_list = (TreeView*) FindControl(L"group_list");
	nim_ui::ContactsListManager::GetInstance()->AttachGroupListBox(group_list);	
}
```

UI组件会自动好友列表的顶端位置显示`创建普通群按钮`、`创建高级群按钮`、`搜索高级群按钮`。单击`创建普通群按钮`会打开`创建普通群组窗口`，单击`创建高级群按钮`会打开`创建高级群组窗口`，单击`搜索高级群按钮`会打开`搜索高级群窗口`。


当集成群组列表的窗体销毁时，应该解除对`TreeView`控件的依附，解除代码如下：

```
nim_ui::ContactsListManager::GetInstance()->AttachGroupListBox(nullptr);
```

### 集成联系人搜索列表

联系人搜索列表组件位于`ContactsListManager`类中。在您需要集成联系人搜索列表的窗口类中，为其对应的XML布局中增加一个`VListBox`控件，在窗口类初始化时获取对应`VListBox`控件的指针，当需要从联系人（包括好友列表和群组列表）中搜索出匹配项并显示的时候，调用`ContactsListManager`类的`FillSearchResultList`函数，传入`VListBox`控件的指针和匹配关键字，就可以得到搜索结果。示例代码如下：

```
bool MainForm::SearchEditChange(ui::EventArgs* param) 
{
	UTF8String search_key = search_edit_->GetUTF8Text();
	bool has_serch_key = !search_key.empty();
	search_result_list_->SetVisible(has_serch_key);

	if (has_serch_key)
	{
		nim_ui::ContactsListManager::GetInstance()->FillSearchResultList(search_result_list_, search_key);
		FindControl(L"no_search_result_tip")->SetVisible(search_result_list_->GetCount() == 0);
	}
	return true;
}
```

### 集成会话窗口

UI组件提供点对点聊天和群聊会话窗口，其中群分为普通群和高级群，高级群提供了更好的群内权限控制功能。会话窗口管理接口位于`SessionManager`类中。  
调用`OpenSessionBox`函数来打开一个会话窗口。打开会话窗口时，需要传入帐号（单聊时为对方个人帐号/群聊时为群号），指定回话类型（P2P/TEAM），是否需要重新打开会话窗口（默认为false），示例如下：

```
nim_ui::SessionManager::GetInstance()->OpenSessionBox(user_account, nim::kNIMSessionTypeP2P);
nim_ui::SessionManager::GetInstance()->OpenSessionBox(team_id, nim::kNIMSessionTypeTeam);
nim_ui::SessionManager::GetInstance()->OpenSessionBox(team_id, nim::kNIMSessionTypeTeam, true);
```

`SessionManager`类中的`IsSessionBoxActive`函数可以判断某个会话盒子是否处于激活状态，示例如下：

```
nim_ui::SessionManager::GetInstance()->IsSessionBoxActive(user_account);
```

`SessionManager`类中的`FindSessionBox`函数可以判断某个会话盒子是否打开，如果打开则返回会话盒子的指针，示例如下：
```
nim_comp::SessionBox *form = nim_ui::SessionManager::GetInstance()->FindSessionBox(user_account);
```

### 个人名片窗口

可以查看当前用户或者好友的名片。如果查看自己的名片，可以设置个人资料和头像等内容；如果查看好友的名片，可以另外设置是否把好友加入黑名单以及是否消息提醒。个人名片窗口接口位于`WindowsManager`类中，在打开个人名片窗口只需传入用户id。示例如下：

```
//打开个人名片
nim_ui::WindowsManager::GetInstance()->ShowProfileForm(nim_ui::LoginManager::GetInstance()->GetAccount());
//打开好友名片
nim_ui::WindowsManager::GetInstance()->ShowProfileForm(uid);
```

### 其他窗口的打开

在UI组件内部还存在许多窗口，多数窗口没有特殊设置，可以直接使用`WindowsManager`类中提供的`SingletonShow`函数打开。需要特殊设置的内部窗口，可以使用`WindowsManager`类中单独提供的函数来打开。当前的`WindowsManager`类中另外提供了打开断线重连的窗口函数`ShowLinkForm`和打开音视频设置的函数`ShowVideoSettingForm`

打开断线重连窗口的示例如下：

```
nim_ui::WindowsManager::GetInstance()->ShowLinkForm();
```

打开音视频设置窗口的示例如下：

```
nim_ui::WindowsManager::GetInstance()->ShowVideoSettingForm();
```

其他窗口都使用`SingletonShow`函数打开，示例如下：

```
//打开添加好友窗口
nim_ui::WindowsManager::GetInstance()->SingletonShow<nim_comp::AddFriendWindow>(nim_comp::AddFriendWindow::kClassName);

//打开黑名单窗口
nim_ui::WindowsManager::GetInstance()->SingletonShow<nim_comp::BlackListWindow>(nim_comp::BlackListWindow::kClassName);

```

### 自定义UI组件界面

UI组件的窗口和界面效果，是由`bin\themes\default`目录中的素材和XML布局配置文件来定义的。  
您可以替换为自己的图片素材来改变界面效果。  
修改XML布局文件可以重新定义UI组件的布局效果。**但是我们只建议您在熟练使用`云信DuiLib`之后再修改XML布局。**

**推荐客户得比特币,首次推荐得0.02BTC,连续推荐得0.03BTC/单,上不封顶。点击参与https://yunxin.163.com/promotion/recommend**

![main](https://github.com/netease-im/NIM_iOS_UIKit/blob/master/800x160.png)
