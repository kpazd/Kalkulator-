import React, { useState, useEffect } from 'react';
import { Calculator, RotateCcw } from 'lucide-react';

const RunningPaceCalculator = () => {
  const [unit, setUnit] = useState('km');
  const [time, setTime] = useState('');
  const [pace, setPace] = useState('');
  const [distance, setDistance] = useState('');
  const [lockedField, setLockedField] = useState(null);
  const [splits, setSplits] = useState([]);
  const [currentPace, setCurrentPace] = useState('');
  
  const [timeDigits, setTimeDigits] = useState('');
  const [paceDigits, setPaceDigits] = useState('');

  const KM_TO_MILE = 0.621371;
  const MILE_TO_KM = 1.60934;

  const presetDistances = {
    km: [
      { label: '5 km', value: 5 },
      { label: '10 km', value: 10 },
      { label: 'Półmaraton', value: 21.097 },
      { label: 'Maraton', value: 42.195 }
    ],
    mile: [
      { label: '3.1 mi', value: 3.1 },
      { label: '6.2 mi', value: 6.2 },
      { label: 'Półmaraton', value: 13.1 },
      { label: 'Maraton', value: 26.2 }
    ]
  };

  const splitDistances = {
    km: [
      { label: '100 m', value: 0.1 },
      { label: '200 m', value: 0.2 },
      { label: '400 m', value: 0.4 },
      { label: '1 km', value: 1 },
      { label: '1500 m', value: 1.5 },
      { label: '5 km', value: 5 },
      { label: '10 km', value: 10 }
    ],
    mile: [
      { label: '100 yd', value: 0.0568 },
      { label: '200 yd', value: 0.1136 },
      { label: '400 yd', value: 0.2273 },
      { label: '1 mi', value: 1 },
      { label: '1.5 mi', value: 1.5 },
      { label: '5 km', value: 3.107 },
      { label: '10 km', value: 6.214 }
    ]
  };

  const formatTimeInput = (digits, isFullTime) => {
    if (digits.length === 0) return '';
    
    if (isFullTime) {
      if (digits.length <= 2) {
        return '0:' + digits.padStart(2, '0');
      } else if (digits.length <= 4) {
        const minutes = digits.slice(0, -2);
        const seconds = digits.slice(-2);
        return minutes + ':' + seconds;
      } else {
        const hours = digits.slice(0, -4);
        const minutes = digits.slice(-4, -2);
        const seconds = digits.slice(-2);
        return hours + ':' + minutes + ':' + seconds;
      }
    } else {
      if (digits.length === 1) {
        return '0:0' + digits;
      } else if (digits.length === 2) {
        return '0:' + digits;
      } else {
        const minutes = digits.slice(0, -2);
        const seconds = digits.slice(-2);
        return minutes + ':' + seconds;
      }
    }
  };

  const handleTimeInput = (value) => {
    const digits = value.replace(/[^0-9]/g, '');
    setTimeDigits(digits);
    const formatted = formatTimeInput(digits, true);
    setTime(formatted);
  };

  const handlePaceInput = (value) => {
    const digits = value.replace(/[^0-9]/g, '');
    setPaceDigits(digits);
    const formatted = formatTimeInput(digits, false);
    setPace(formatted);
  };

  useEffect(() => {
    const filledFields = [time, pace, distance].filter(f => f !== '' && f !== '0:00').length;
    
    if (filledFields === 2) {
      if (!time || time === '0:00') setLockedField('time');
      else if (!pace || pace === '0:00') setLockedField('pace');
      else if (!distance) setLockedField('distance');
    } else {
      setLockedField(null);
    }
  }, [time, pace, distance]);

  const timeToSeconds = (timeStr) => {
    if (!timeStr) return 0;
    const parts = timeStr.split(':').map(p => parseInt(p) || 0);
    if (parts.length === 3) {
      return parts[0] * 3600 + parts[1] * 60 + parts[2];
    } else if (parts.length === 2) {
      return parts[0] * 60 + parts[1];
    }
    return parts[0] || 0;
  };

  const secondsToTime = (seconds) => {
    const h = Math.floor(seconds / 3600);
    const m = Math.floor((seconds % 3600) / 60);
    const s = Math.floor(seconds % 60);
    
    if (h > 0) {
      return h + ':' + m.toString().padStart(2, '0') + ':' + s.toString().padStart(2, '0');
    }
    return m + ':' + s.toString().padStart(2, '0');
  };

  const formatSplitTime = (seconds) => {
    const h = Math.floor(seconds / 3600);
    const m = Math.floor((seconds % 3600) / 60);
    const s = Math.floor(seconds % 60);
    
    if (h > 0) {
      return h + 'h ' + m + 'min ' + s + 's';
    } else if (m > 0) {
      return m + 'min ' + s + 's';
    } else {
      return s + 's';
    }
  };

  const handleUnitChange = (newUnit) => {
    if (newUnit === unit) return;
    
    const conversionFactor = newUnit === 'mile' ? KM_TO_MILE : MILE_TO_KM;
    
    if (distance) {
      const distValue = parseFloat(distance);
      if (!isNaN(distValue)) {
        setDistance((distValue * conversionFactor).toFixed(2));
      }
    }
    
    if (pace) {
      const paceSeconds = timeToSeconds(pace);
      const newPaceSeconds = paceSeconds / conversionFactor;
      const newPaceStr = secondsToTime(newPaceSeconds);
      setPace(newPaceStr);
      
      if (splits.length > 0) {
        setCurrentPace(newPaceStr);
        calculateSplits(newPaceSeconds, newUnit);
      }
    }
    
    setUnit(newUnit);
  };

  const calculateSplits = (paceSeconds, unitToUse) => {
    const useUnit = unitToUse || unit;
    const newSplits = splitDistances[useUnit].map(split => {
      const totalSeconds = paceSeconds * split.value;
      return {
        label: split.label,
        time: formatSplitTime(totalSeconds)
      };
    });
    setSplits(newSplits);
  };

  const calculate = () => {
    try {
      let finalPaceSeconds = null;
      let finalPaceString = '';

      if (lockedField === 'time') {
        const paceSeconds = timeToSeconds(pace);
        const dist = parseFloat(distance);
        const totalSeconds = paceSeconds * dist;
        const resultTime = secondsToTime(totalSeconds);
        setTime(resultTime);
        finalPaceSeconds = paceSeconds;
        finalPaceString = pace;
      } else if (lockedField === 'pace') {
        const timeSeconds = timeToSeconds(time);
        const dist = parseFloat(distance);
        const paceSeconds = timeSeconds / dist;
        const resultPace = secondsToTime(paceSeconds);
        setPace(resultPace);
        finalPaceSeconds = paceSeconds;
        finalPaceString = resultPace;
      } else if (lockedField === 'distance') {
        const timeSeconds = timeToSeconds(time);
        const paceSeconds = timeToSeconds(pace);
        const dist = timeSeconds / paceSeconds;
        setDistance(dist.toFixed(2));
        finalPaceSeconds = paceSeconds;
        finalPaceString = pace;
      }

      if (finalPaceSeconds) {
        setCurrentPace(finalPaceString);
        calculateSplits(finalPaceSeconds, unit);
      }
    } catch (error) {
      alert('Błąd w obliczeniach. Sprawdź wprowadzone dane.');
    }
  };

  const clearAll = () => {
    setTime('');
    setPace('');
    setDistance('');
    setTimeDigits('');
    setPaceDigits('');
    setLockedField(null);
    setSplits([]);
    setCurrentPace('');
  };

  const handleDistancePreset = (value) => {
    if (value) {
      setDistance(value);
    }
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-3 sm:p-4">
      <div className="max-w-md mx-auto">
        <div className="bg-white rounded-2xl shadow-xl p-4 sm:p-6 mb-4">
          <div className="flex flex-col sm:flex-row justify-between items-center gap-3 mb-4 sm:mb-6">
            <h1 className="text-2xl sm:text-3xl font-bold text-gray-800 flex items-center gap-2">
              <Calculator className="text-indigo-600" size={28} />
              Kalkulator Tempa
            </h1>
            <button
              onClick={clearAll}
              className="flex items-center gap-2 px-4 py-2 bg-red-500 text-white rounded-lg hover:bg-red-600 transition text-sm sm:text-base"
            >
              <RotateCcw size={18} />
              Wyczyść
            </button>
          </div>

          <div className="flex justify-center gap-2 mb-4 sm:mb-6">
            <button
              onClick={() => handleUnitChange('km')}
              className={'flex-1 px-4 sm:px-6 py-2.5 sm:py-2 rounded-lg font-semibold transition text-base sm:text-base ' + (unit === 'km' ? 'bg-indigo-600 text-white' : 'bg-gray-200 text-gray-700 hover:bg-gray-300')}
            >
              km
            </button>
            <button
              onClick={() => handleUnitChange('mile')}
              className={'flex-1 px-4 sm:px-6 py-2.5 sm:py-2 rounded-lg font-semibold transition text-base sm:text-base ' + (unit === 'mile' ? 'bg-indigo-600 text-white' : 'bg-gray-200 text-gray-700 hover:bg-gray-300')}
            >
              mile
            </button>
          </div>

          <div className="space-y-3 sm:space-y-4 mb-4 sm:mb-6">
            <div>
              <label className="block text-sm font-semibold text-gray-700 mb-1.5 sm:mb-2">
                Czas (hh:mm:ss)
              </label>
              <input
                type="text"
                inputMode="numeric"
                value={time}
                onChange={(e) => handleTimeInput(e.target.value)}
                onFocus={(e) => {
                  if (time === '0:00') {
                    setTime('');
                    setTimeDigits('');
                  }
                }}
                disabled={lockedField === 'time'}
                placeholder="np. wpisz 4530 → 45:30"
                className={'w-full px-3 sm:px-4 py-3 border-2 rounded-lg text-base sm:text-lg ' + (lockedField === 'time' ? 'bg-gray-100 border-gray-300 text-gray-500 cursor-not-allowed' : 'border-indigo-300 focus:border-indigo-500 focus:outline-none')}
              />
            </div>

            <div>
              <label className="block text-sm font-semibold text-gray-700 mb-1.5 sm:mb-2">
                Tempo (min/{unit})
              </label>
              <input
                type="text"
                inputMode="numeric"
                value={pace}
                onChange={(e) => handlePaceInput(e.target.value)}
                onFocus={(e) => {
                  if (pace === '0:00') {
                    setPace('');
                    setPaceDigits('');
                  }
                }}
                disabled={lockedField === 'pace'}
                placeholder="np. wpisz 445 → 4:45"
                className={'w-full px-3 sm:px-4 py-3 border-2 rounded-lg text-base sm:text-lg ' + (lockedField === 'pace' ? 'bg-gray-100 border-gray-300 text-gray-500 cursor-not-allowed' : 'border-indigo-300 focus:border-indigo-500 focus:outline-none')}
              />
            </div>

            <div>
              <label className="block text-sm font-semibold text-gray-700 mb-1.5 sm:mb-2">
                Dystans ({unit})
              </label>
              <div className="flex gap-2">
                <input
                  type="text"
                  value={distance}
                  onChange={(e) => {
                    const value = e.target.value.replace(/[^0-9.]/g, '');
                    setDistance(value);
                  }}
                  disabled={lockedField === 'distance'}
                  placeholder="np. 10"
                  className={'flex-1 px-3 sm:px-4 py-3 border-2 rounded-lg text-base sm:text-lg ' + (lockedField === 'distance' ? 'bg-gray-100 border-gray-300 text-gray-500 cursor-not-allowed' : 'border-indigo-300 focus:border-indigo-500 focus:outline-none')}
                />
                <select
                  onChange={(e) => handleDistancePreset(e.target.value)}
                  value=""
                  disabled={lockedField === 'distance'}
                  className="px-2 sm:px-3 py-3 border-2 border-indigo-300 rounded-lg focus:border-indigo-500 focus:outline-none disabled:bg-gray-100 disabled:border-gray-300 text-base"
                >
                  <option value="">▼</option>
                  {presetDistances[unit].map((preset, idx) => (
                    <option key={idx} value={preset.value}>
                      {preset.label}
                    </option>
                  ))}
                </select>
              </div>
            </div>
          </div>

          <button
            onClick={calculate}
            disabled={lockedField === null}
            className={'w-full py-3.5 sm:py-4 rounded-lg font-bold text-base sm:text-lg transition ' + (lockedField ? 'bg-indigo-600 text-white hover:bg-indigo-700 cursor-pointer' : 'bg-gray-300 text-gray-500 cursor-not-allowed')}
          >
            OBLICZ
          </button>
        </div>

        {splits.length > 0 && currentPace && (
          <div className="bg-white rounded-2xl shadow-xl p-4 sm:p-6">
            <h2 className="text-xl sm:text-2xl font-bold text-gray-800 mb-3 sm:mb-4">
              Tempo {currentPace} min/{unit}
            </h2>
            <div className="space-y-2">
              {splits.map((split, idx) => (
                <div
                  key={idx}
                  className="flex justify-between items-center p-3 bg-indigo-50 rounded-lg"
                >
                  <span className="font-semibold text-gray-700 text-base">
                    {split.label}
                  </span>
                  <span className="text-indigo-600 font-mono font-bold text-base sm:text-lg">
                    {split.time}
                  </span>
                </div>
              ))}
            </div>
          </div>
        )}
      </div>
    </div>
  );
};

export default RunningPaceCalculator;
