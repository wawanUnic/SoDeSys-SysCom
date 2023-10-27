# SoDeSys-SysCom
Пример работы с библиотекой SysCom с автоматическим склеиванием пакетов.

PROGRAM main
VAR
	eStep, eStepError : eSteps := eSteps.openPort;
	hCom : SysCom.RTS_IEC_HANDLE; // Указатель на порт
	stPortSettings : SysCom.SysComSettings;
	portResult : SysCom.RTS_IEC_RESULT; // Результат функций
	arrSend : ARRAY [1..255] OF BYTE :=	[16#C0, 16#05, 16#00, 16#FF, 16#FF, 16#FF, 16#07, 16#03, 16#00,
										16#04, 16#02, 16#80, 16#00, 16#02, 16#81, 16#00, 16#02, 16#84, 16#8E, 16#02, 16#83, 16#7F,
										16#F4, 16#01, 16#05, 16#00,
										16#0A, 16#C4, 16#7B, 16#00,
										16#BF, 16#19, 16#C0 ];
	arrReceive, arrReceiveLater : ARRAY [1..255] OF BYTE;
	udiLen, udiLenLater : UDINT;
END_VAR

// !!! Требуется библиотека SysCom !!!

CASE eStep OF
	
	eSteps.openPort:
						// Открытие порта
						stPortSettings.sPort := SysCom.SYS_COM_PORTS.SYS_COMPORT3;							// Имя порта
						stPortSettings.byStopBits := SysCom.SYS_COM_STOPBITS.SYS_ONESTOPBIT;				// Стопы
						stPortSettings.byParity := SysCom.SYS_COM_PARITY.SYS_NOPARITY;						// Паритет
						stPortSettings.ulBaudrate := sysCom.SYS_COM_BAUDRATE.SYS_BR_115200;					// Битрейд
						stPortSettings.ulTimeout := sysCom.SYS_COM_TIMEOUT.SYS_NOWAIT;
						stPortSettings.ulBufferSize := 255;													// Размер буфера FIFO
						
						hCom := SysCom.SysComOpen2(	pSettings := ADR(stPortSettings),						// Возвращает указатель на порт
															pSettingsEx := 0,								// Должно быть 0
															pResult := ADR(portResult)); 
						IF portResult = CmpErrors.Errors.ERR_OK THEN
							eStep := eSteps.writePort;
						ELSE
							eStep := eSteps.error;
						END_IF

	
	eSteps.writePort:
						// Отправка в порт
						udiLen := SysCom.SysComWrite(		hCom := hCom,									// Возвращает кол-во отправленных байт
															pbyBuffer := ADR(arrSend),
															ulSize := 33,
															ulTimeout := 0, // mS
															pResult := portResult);
						IF portResult = CmpErrors.Errors.ERR_OK THEN
							eStep := eSteps.readPort;
						ELSE
							eStep := eSteps.error;
						END_IF
	
	eSteps.readPort:
						// Чтение из порта
						udiLen := SysCom.SysComRead(		hCom := hCom,									// Возвращает кол-во прочитанных байт
															pbyBuffer := ADR(arrReceive),
															ulSize := 255,
															ulTimeout := 0, // mS
															pResult := portResult);
						IF portResult <> CmpErrors.Errors.ERR_OK THEN
							eStep := eSteps.error;
						ELSIF udiLen <> 0 THEN
							SysCom.SysComPurge(hCom);
							eStep := eSteps.readPortLater;
						ELSE
							; //
						END_IF
						
	eSteps.readPortLater:
						// Дочтение из порта
						udiLenLater := SysCom.SysComRead(		hCom := hCom,								// Возвращает кол-во дочитанных байт
																pbyBuffer := ADR(arrReceiveLater),
																ulSize := 255,
																ulTimeout := 0, // mS
																pResult := portResult);
						IF portResult <> CmpErrors.Errors.ERR_OK THEN
							eStep := eSteps.error;
						ELSIF udiLenLater <> 0 THEN
							MEM.MemMove(	ADR(arrReceiveLater),											// Склеивание
											ADR(arrReceive) + udiLen,
											UDINT_TO_UINT(udiLenLater));
							SysCom.SysComPurge(hCom);
							eStep := eSteps.success;
						ELSE
							SysCom.SysComPurge(hCom);
							eStep := eSteps.success;
						END_IF
						
	eSteps.success:
						// Успешно
						;
	
	eSteps.error:
						// Ошибка
						;
						
END_CASE
