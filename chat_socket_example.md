```
// Dart imports:
import 'dart:async';
import 'dart:convert';
import 'dart:io';

// Package imports:
import 'package:socket_io_client/socket_io_client.dart' as socket_io;

class ChatSocket {
  static socket_io.Socket? socket;
  //* Socket Init
  static void initSocket() async {
    if (!(socket?.connected ?? false)) {
      String userToken = await PreferenceServiceHelper().getStringPrefValue(key: PreferenceKeys.keyAccessToken);
      LogHelper.logSuccess("user token : $userToken", stackTrace: StackTrace.current);
      socket = socket_io.io(EnvConfig().onlineSocketUrl, {
        "transports": ['websocket'],
        "autoConnect": true,
        "reconnection": true,
        "extraHeaders": {HttpHeaders.authorizationHeader: userToken}
      });

      /// on error
      socket?.on('error', (data) => LogHelper.logError("socket error $data"));

      /// on connect
      socket?.onConnect((_) {
        LogHelper.logSuccess("Socket Connected");
        String? chatID = NavigationService.context.read<ChatProvider>().currentChatId;
        if (chatID?.isNotEmpty ?? false) {
          eventJoinChannelEmit(chatId: chatID ?? '');
          eventMessageListen();
        }
      });

      /// on disconnect
      socket?.onDisconnect((_) {
        LogHelper.logError(
            '------------------------------------- \nSocket Disconnected\n -------------------------------------');
        socket?.off(SocketEvent.MESSAGE.value);
      });
    }

    /// on user chat listen
    eventUserChatsListen();
  }

//!=========
//!Listeners
//!=========

//* offline user count on listen
  static void eventOfflineUserCountListen() async {
    bool? hasListeners = socket?.hasListeners(SocketEvent.OFFLINE_USER_COUNTS.value);

    if (!(hasListeners ?? false)) {
      LogHelper.logInfo("sck : eventLiveChatCountListen");
      socket?.on(SocketEvent.OFFLINE_USER_COUNTS.value, (data) {
        LogHelper.logSuccess("on ${SocketEvent.OFFLINE_USER_COUNTS.value} : $data");
        List<OfflineUserCountResModel> model = [];
        for (var i = 0; i < data.length; i++) {
          model.add(OfflineUserCountResModel.fromJson(json.decode(Utils.getPrettyJSONString(data[i]))));
        }

        NavigationService.context.read<ChatProvider>().setLastActiveTimeFromSocket(model: model);
      });
    }
  }

  //* live chat count on listen
  static void eventLiveChatCountListen() async {
    bool? hasListeners = socket?.hasListeners(SocketEvent.LIVE_CHAT_COUNTS.value);

    if (!(hasListeners ?? false)) {
      LogHelper.logInfo("sck : eventLiveChatCountListen");
      socket?.on(SocketEvent.LIVE_CHAT_COUNTS.value, (data) {
        LogHelper.logSuccess("on ${SocketEvent.LIVE_CHAT_COUNTS.value} : $data");
        List<LiveChatsCountResModel> model = [];
        for (var i = 0; i < data.length; i++) {
          model.add(LiveChatsCountResModel.fromJson(json.decode(Utils.getPrettyJSONString(data[i]))));
        }

        NavigationService.context.read<ChatProvider>().setLiveChatCountFromSocket(model: model);
      });
    }
  }

  //* message on listen
  static void eventMessageListen() async {
    bool? hasListeners = socket?.hasListeners(SocketEvent.MESSAGE.value);

    if (!(hasListeners ?? false)) {
      socket?.on(SocketEvent.MESSAGE.value, (data) async {
        LogHelper.logSuccess("on ${SocketEvent.MESSAGE.value} : $data");
        OnMessageResModel model = OnMessageResModel.fromJson(json.decode(Utils.getPrettyJSONString(data)));
        LogHelper.logInfo("sck : eventMessageListen ${model.sId}");
        NavigationService.context.read<MessageProvider>().setMessageFromSocket(model: model);

        if (!NavigationService.context.read<MessageProvider>().isChatBlocked(model.chatId ?? "")) {
          // eventReadMessagesEmit(chatId: model.chatId ?? "");
        }
      });
    }
  }

  //* listen on chat list.
  static void eventUserChatsListen() {
    bool? hasListeners = socket?.hasListeners(SocketEvent.USER_CHATS.value);
    if (!(hasListeners ?? false)) {
      socket?.on(SocketEvent.USER_CHATS.value, (data) {
        LogHelper.logSuccess("on ${SocketEvent.USER_CHATS.value} : $data");
        ChatUsersEventResModel chatUsersEventResModel =
            ChatUsersEventResModel.fromJson(json.decode(Utils.getPrettyJSONString(data)));
        NavigationService.context.read<ChatProvider>().setChatDetails(chatUsersEventModel: chatUsersEventResModel);
      });
    }
  }

  //* listen read messages.
  static void eventReadMessagesListen() {
    bool? hasListeners = socket?.hasListeners(SocketEvent.READ_MESSAGES.value);
    if (!(hasListeners ?? false)) {
      socket?.on(SocketEvent.READ_MESSAGES.value, (data) {
        LogHelper.logInfo("sck : eventReadMessagesListen ${data}");
        LogHelper.logSuccess("on ${SocketEvent.READ_MESSAGES.value} : $data");
        final res = ChatReadResModel.fromJson(data);
        NavigationService.context.read<MessageProvider>().readMessageSocketEventHandler(response: res);
      });
    }
  }

  //* listen deliver messages.
  static void eventDeliverChatListen() {
    bool? hasListeners = socket?.hasListeners(SocketEvent.DELIVER_MESSAGE.value);
    if (!(hasListeners ?? false)) {
      socket?.on(SocketEvent.DELIVER_MESSAGE.value, (data) {
        LogHelper.logInfo("sck : eventDeliverChatListen");
        LogHelper.logSuccess("on ${SocketEvent.DELIVER_MESSAGE.value} : $data");
        final res = ChatDeliveredResModel.fromJson(data);
        NavigationService.context.read<MessageProvider>().deliverMessageSocketEventHandler(response: res);
      });
    }
  }

  //* listen delete messages.
  static void eventDeleteMessageListen() {
    bool? hasListeners = socket?.hasListeners(SocketEvent.DELETE_MESSAGE.value);
    if (!(hasListeners ?? false)) {
      socket?.on(SocketEvent.DELETE_MESSAGE.value, (data) {
        LogHelper.logInfo("sck : eventDeleteChatListen");
        LogHelper.logSuccess("on ${SocketEvent.DELETE_MESSAGE.value} : $data");
        final response = DeleteMessagesReqModel.fromJson(json.decode(data) as Map<String, dynamic>);
        NavigationService.context.read<MessageProvider>().deleteMessageSocketEventHandler(response: response);
      });
    }
  }

  //* listen live location.
  // static void eventLiveLocationListen() {
  //   bool? hasListeners = socket?.hasListeners(SocketEvent.LIVE_LOCATION.value);
  //   if (!(hasListeners ?? false)) {
  //     LogHelper.logInfo("sck : eventLiveLocationListen");
  //     socket?.on(SocketEvent.LIVE_LOCATION.value, (data) {
  //       LogHelper.logSuccess("on ${SocketEvent.LIVE_LOCATION.value} : $data");
  //       var response;
  //       try {
  //         response = LiveLocationResModel.fromJson(json.decode(data.toString()));
  //       } catch (e) {
  //         response = LiveLocationResModel.fromJson(json.decode(Utils.getPrettyJSONString(data)));
  //       }
  //       NavigationService.context.read<MessageProvider>().setLocationFromSocket(response);
  //     });
  //   }
  // }

  //* listen message user created.
  static void eventMessageUserCreatedListen() {
    bool? hasListeners = socket?.hasListeners(SocketEvent.MESSAGE_USER_CREATED.value);
    if (!(hasListeners ?? false)) {
      LogHelper.logInfo("sck : eventLiveLocationListen");
      socket?.on(SocketEvent.MESSAGE_USER_CREATED.value, (data) {
        LogHelper.logSuccess("on ${SocketEvent.MESSAGE_USER_CREATED.value} : $data");
        // final response = LiveLocationResModel.fromJson(json.decode(data.toString()));
        // NavigationService.context.read<MessageProvider>().setLocationFromSocket(response);
      });
    }
  }

//!=========
//!Emit
//!=========
  //* Send message
  static void eventSendMessageEmit({required SendMessageReqSocketModel model}) {
    LogHelper.logCyan(json.encode(model.toJson()), stackTrace: StackTrace.current);
    socket?.emit(SocketEvent.MESSAGE.value, json.encode(model.toJson()));
  }

  //* join chat.
  static void eventJoinChannelEmit({required String chatId, bool isBlocked = false}) {
    LogHelper.logInfo("sck : eventJoinChannelEmit : $chatId");
    socket?.emit(SocketEvent.JOIN_CHANNEL.value, chatId);
    eventMessageListen();
    eventReadMessagesListen();
    eventDeliverChatListen();
    eventDeleteMessageListen();
    // eventLiveLocationListen();
    eventMessageUserCreatedListen();
    if (!isBlocked) {
      eventReadMessagesEmit(chatId: chatId);
    }
  }

  //* leave chat.
  static void eventLeaveChannelEmit({required String chatId}) {
    LogHelper.logInfo("sck : eventLeaveChannelEmit : $chatId");
    socket?.emit(SocketEvent.LEAVE_CHANNEL.value, chatId);

    _eventMessageOff();
    _eventReadMessagesOff();
    _eventDeleteMessagesOff();
    _eventDeliverMessagesOff();
    _eventLiveLocationOff();
  }

  //* read messages.
  static Future<void> eventReadMessagesEmit({required String chatId}) async {
    LogHelper.logSuccess("sck : eventReadMessagesEmit : Timer started");
    await Future.delayed(const Duration(milliseconds: 500));
    LogHelper.logSuccess("sck : eventReadMessagesEmit : Timer completed");
    socket?.emit(SocketEvent.READ_MESSAGES.value, chatId);
    LogHelper.logInfo("sck : eventReadMessagesEmit : $chatId");
  }

  //* update live location
  // static void eventUpdateLocationEmit({required UpdateLiveLocationReqModel model}) {
  //   LogHelper.logInfo("sck : eventUpdateLocationEmit : ${json.encode(model.toJson())}", stackTrace: StackTrace.current);
  //   socket?.emit(SocketEvent.LIVE_LOCATION.value, json.encode(model.toJson()));
  // }

  //** Delete Message emit
  static void eventDeleteMessagesEmit({required DeleteMessagesReqModel messageIds}) {
    LogHelper.logSuccess("sck : eventDeleteMessagesEmit :::${messageIds.toJson()}");
    NavigationService.context
        .read<MessageProvider>()
        .deleteMessageSocketEventHandler(response: DeleteMessagesReqModel(msgIds: messageIds.msgIds ?? []));
    socket?.emit(SocketEvent.DELETE_MESSAGE.value, json.encode(messageIds.toJson()));
  }

  static void disconnectSocket() {
    socket?.disconnect();
    socket?.dispose();
  }

  //!
  //! off socket
  //!

  static void leaveAllSocket() {
    _eventMessageOff();
    _eventReadMessagesOff();
    _eventDeleteMessagesOff();
    _eventDeliverMessagesOff();
    _eventLiveLocationOff();
  }

  static void _eventMessageOff() {
    LogHelper.logInfo("sck : eventMessageOff");
    socket?.off(SocketEvent.MESSAGE.value);
  }

  static void _eventLiveLocationOff() {
    LogHelper.logInfo("sck : _eventLiveLocationOff");
    socket?.off(SocketEvent.LIVE_LOCATION.value);
  }

  static void _eventReadMessagesOff() {
    LogHelper.logInfo("sck : _eventReadMessagesOff");
    socket?.off(SocketEvent.READ_MESSAGES.value);
  }

  static void _eventDeleteMessagesOff() {
    LogHelper.logInfo("sck : _eventDeleteMessagesOff");
    socket?.off(SocketEvent.DELETE_MESSAGE.value);
  }

  static void _eventDeliverMessagesOff() {
    LogHelper.logInfo("sck : _eventDeleteMessagesOff");
    socket?.off(SocketEvent.DELIVER_MESSAGE.value);
  }
}


```
